// =============================================================================
// Jenkinsfile — Wave-based deploy to Anypoint Flex Gateway
//               Driven by flex-gateway-config repo (GitOps)
// -----------------------------------------------------------------------------
// Flow:
//   1. Checkout flex-gateway-config repo (or the repo this Jenkinsfile lives in)
//   2. Read anypoint.yaml → resolve orgId, envId, flexTarget for chosen ENVIRONMENT
//   3. Read waves/<WAVE>/manifest.yaml → list of apps to deploy
//   4. For each app: read apis/<app>/config.yaml + envs/<env>/overlay.yaml
//   5. Authenticate via Connected App (client_credentials)
//   6. Deploy each wave in order:
//        find-or-create instance → deploy → apply policies → validate
//   7. Print summary; fail build if any API failed
//
// Config (non-secret) comes from the flex-gateway-config Git repo.
// Secrets come from the Jenkins credentials store — never from the repo.
//
// Required Jenkins plugins : Pipeline Utility Steps (readYaml, readJSON, writeJSON)
// Required agent tools     : git, curl, python3
// Required credentials     : anypoint-connected-app-<env>
//                             (Kind: Username+Password, user=CLIENT_ID, pass=CLIENT_SECRET)
// =============================================================================

pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT',
               choices: ['dev', 'test', 'qa', 'preprod', 'prod','sandbox'],
               description: 'Target Anypoint environment')
        string(name: 'WAVE',
               defaultValue: 'R1',
               description: 'Wave to run: R1, R2, R3, R4, or ALL. Case-insensitive.')
        string(name: 'APP_FILTER',
               defaultValue: '',
               description: 'Optional: comma/space-separated app names to limit within the wave (e.g. "payments-api,orders-api"). Blank = all apps in the wave.')
        booleanParam(name: 'DRY_RUN',
               defaultValue: true,
               description: 'Print the deployment plan without calling Anypoint APIs. Default true — set false to actually deploy.')
        booleanParam(name: 'SKIP_POLICIES',
               defaultValue: false,
               description: 'Skip policy application step (useful for initial connectivity testing)')
    }

    environment {
        CORRELATION_ID        = "${env.BUILD_TAG}"
        MULESOFT_POLICY_GROUP = '68ef9520-24e9-4cf2-b2f5-620025690913'
        HTTPS_PROXY           = 'http://proxy.infosec.fedex.com:443'
        HTTP_PROXY            = 'http://proxy.infosec.fedex.com:443'
        NO_PROXY              = 'localhost,127.0.0.1'
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        // ──────────────────────────────────────────────────────────────────── //
        stage('Checkout Config') {
            steps {
                script {
                    log('INFO', 'Checking out the repo this Jenkinsfile lives in (checkout scm)')
                    checkout scm
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Load Config') {
            steps {
                script {
                    // ── Anypoint platform config ───────────────────────────────
                    if (!fileExists('anypoint.yaml')) {
                        error 'anypoint.yaml not found in the config repo root. Create it from the setup guide.'
                    }
                    def acfg     = readYaml file: 'anypoint.yaml'
                    def envs     = acfg.anypoint?.environments ?: [:]
                    // Case-insensitive lookup so 'dev', 'DEV', 'Dev' all match
                    def envKey   = envs.keySet().find { it.equalsIgnoreCase(params.ENVIRONMENT) }
                    def envBlock = envKey ? envs[envKey] : null
                    if (!envBlock) {
                        error "No anypoint config found for environment '${params.ENVIRONMENT}' in anypoint.yaml (available: ${envs.keySet().join(', ')})"
                    }
                    env.ORG_ID            = "${acfg.anypoint.organizationId}"
                    env.ENV_ID            = "${envBlock.environmentId}"
                    env.GATEWAY_VERSION   = "${envBlock.gatewayVersion}"
                    env.FLEX_TARGET_ID    = "${envBlock.flexTarget.id}"
                    env.FLEX_TARGET_NAME  = "${envBlock.flexTarget.name}"
                    env.ANYPOINT_BASE_URL = "${acfg.anypoint.baseUrl ?: 'https://anypoint.mulesoft.com'}"

                    log('INFO', "Target: env=${params.ENVIRONMENT} flexTarget=${env.FLEX_TARGET_NAME}")

                    // ── Determine waves ────────────────────────────────────────
                    def waveInput = params.WAVE?.trim()?.toUpperCase() ?: 'R1'
                    def wavesToRun = waveInput == 'ALL'
                        ? ['R1', 'R2', 'R3', 'R4'].findAll { fileExists("waves/${it}/manifest.yaml") }
                        : [waveInput]
                    log('INFO', "Waves to run: ${wavesToRun.join(', ')}")

                    // ── Optional app-level filter ──────────────────────────────
                    def appFilter = params.APP_FILTER?.trim()
                        ? params.APP_FILTER.split('[,\\s]+').collect { it.trim() }.findAll { it }
                        : []

                    // ── Build wave → api-list map ──────────────────────────────
                    // Policy asset-id normalisation: use config.yaml's assetId directly.
                    // rate-limiting (simple, uses rateLimits[]) != rate-limiting-sla (SLA-tier).
                    def POLICY_ID_NORM = [:]

                    // ── Common policies (applied to every API unless overridden) ──
                    def commonPolicies = []
                    if (fileExists('common/policies.yaml')) {
                        def commonCfg = readYaml file: 'common/policies.yaml'
                        commonPolicies = (commonCfg.policies ?: []).collect { p ->
                            [
                                assetId      : "${POLICY_ID_NORM[p.assetId] ?: p.assetId}",
                                groupId      : "${p.groupId ?: env.MULESOFT_POLICY_GROUP}",
                                policyVersion: "${p.policyVersion}",
                                config       : p.config ?: [:]
                            ]
                        }
                        if (commonPolicies) {
                            log('INFO', "Common policies: ${commonPolicies.collect { it.assetId }.join(', ')}")
                        }
                    }

                    def waveApiMap = [:]
                    wavesToRun.each { wave ->
                        def manifestPath = "waves/${wave}/manifest.yaml"
                        if (!fileExists(manifestPath)) {
                            log('WARN', "Wave ${wave}: manifest not found at ${manifestPath} — skipping")
                            return
                        }
                        def manifest = readYaml file: manifestPath
                        // .toString() avoids GString vs String mismatch in List.contains()
                        def apps = (manifest.apps ?: []).collect { it.toString() }

                        if (appFilter) {
                            def filterLower = appFilter.collect { it.toLowerCase() }
                            apps = apps.findAll { filterLower.contains(it.toLowerCase()) }
                            log('INFO', "Wave ${wave}: filtered to [${apps.join(', ')}]")
                        }

                        // Eagerly flatten overlay apps into a plain Groovy Map so that
                        // hyphenated keys (e.g. customer-api) are safe to look up inside
                        // closures under Jenkins CPS serialisation.
                        def overlayApps = [:]
                        def overlayPath = "envs/${params.ENVIRONMENT}/overlay.yaml"
                        if (fileExists(overlayPath)) {
                            def rawOverlay = readYaml file: overlayPath
                            rawOverlay?.apps?.each { k, v -> overlayApps["${k}"] = v }
                        }
                        if (overlayApps) {
                            log('INFO', "Overlay apps found: ${overlayApps.keySet().join(', ')}")
                        } else {
                            log('WARN', "No overlay apps found for ${params.ENVIRONMENT} — proxy ports will default to 8080")
                        }

                        def apiList = []
                        apps.each { appName ->
                            def cfgPath = "apis/${appName}/config.yaml"
                            if (!fileExists(cfgPath)) {
                                error "API config not found: ${cfgPath}"
                            }
                            def apiCfg     = readYaml file: cfgPath
                            def appOverlay = overlayApps[appName] ?: [:]

                            // upstreamUri: env overlay wins, then first endpoint's backend URI
                            def upstreamUri = "${appOverlay.upstreamUri ?: apiCfg.endpoints[0].uri}"
                            // proxyUri: env overlay wins, then a sane default on port 8080
                            def proxyUri    = "${appOverlay.proxyUri    ?: 'http://0.0.0.0:8080'}"
                            log('INFO', "  ${appName}: proxyUri=${proxyUri}  upstream=${upstreamUri}")

                            def apiPolicies = (apiCfg.policies ?: []).collect { p ->
                                [
                                    assetId      : "${POLICY_ID_NORM[p.assetId] ?: p.assetId}",
                                    groupId      : "${p.groupId ?: env.MULESOFT_POLICY_GROUP}",
                                    policyVersion: "${p.policyVersion}",
                                    config       : p.config ?: [:]
                                ]
                            }
                            // Merge: API-specific policies take precedence; common policies fill in the rest
                            def apiSpecificIds = apiPolicies.collect { it.assetId } as Set
                            def policies = apiPolicies + commonPolicies.findAll { !(it.assetId in apiSpecificIds) }

                            apiList << [
                                appName      : "${appName}",
                                assetId      : "${apiCfg.assetId}",
                                assetVersion : "${apiCfg.version}",
                                instanceLabel: "${appName}-${params.ENVIRONMENT}",
                                upstreamUri  : upstreamUri,
                                proxyUri     : proxyUri,
                                policies     : policies,
                                wave         : wave,
                                endpoints    : (apiCfg.endpoints ?: []).collect { ep ->
                                    [name: "${ep.name}", uri: "${ep.uri}", publicPath: "${ep.publicPath ?: ''}"]
                                }
                            ]
                        }
                        waveApiMap[wave] = apiList
                        log('INFO', "Wave ${wave}: [${apiList.collect { it.appName }.join(', ')}]")
                    }

                    WAVES_TO_RUN  = wavesToRun
                    WAVE_API_MAP  = waveApiMap
                    ALL_RESULTS   = []
                    API_INSTANCES = [:]

                    if (params.DRY_RUN) {
                        wavesToRun.each { wave ->
                            (waveApiMap[wave] ?: []).each { api ->
                                log('INFO', "[DRY_RUN] wave=${wave}  app=${api.appName}  ${api.assetId}:${api.assetVersion}")
                                log('INFO', "[DRY_RUN]   upstream=${api.upstreamUri}  proxy=${api.proxyUri}")
                                log('INFO', "[DRY_RUN]   policies=${api.policies?.collect { it.assetId }}")
                                ALL_RESULTS << [wave: wave, label: api.instanceLabel, status: 'DRY_RUN', instanceId: '-']
                            }
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Authenticate to Anypoint') {
            when { expression { !params.DRY_RUN } }
            steps {
                withCredentials([usernamePassword(
                        credentialsId: "anypoint-connected-app-${params.ENVIRONMENT}",
                        usernameVariable: 'CLIENT_ID',
                        passwordVariable: 'CLIENT_SECRET')]) {
                    script {
                        log('INFO', 'Requesting OAuth2 access token via client_credentials grant')
                        def token = sh(
                            script: '''
                                set -e
                                BODY='{"grant_type":"client_credentials","client_id":"'"$CLIENT_ID"'","client_secret":"'"$CLIENT_SECRET"'"}'
                                TMP=$(mktemp)
                                HTTP=$(curl -s -o "$TMP" -w "%{http_code}" -X POST \
                                    "$ANYPOINT_BASE_URL/accounts/api/v2/oauth2/token" \
                                    -H "Content-Type: application/json" \
                                    -d "$BODY")
                                RESP=$(cat "$TMP"); rm -f "$TMP"
                                if [ -z "$RESP" ]; then
                                    echo "ERROR: curl returned empty response (HTTP $HTTP) — check network/proxy connectivity to $ANYPOINT_BASE_URL" >&2
                                    exit 1
                                fi
                                echo "$RESP" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))"
                            ''',
                            returnStdout: true
                        ).trim()
                        if (!token || token == 'null') { error 'Authentication failed: no access_token returned' }
                        env.ANYPOINT_TOKEN = token
                        log('INFO', 'Access token acquired')
                    }
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Publish to Exchange') {
            when { expression { !params.DRY_RUN } }
            steps {
                script {
                    WAVES_TO_RUN.each { wave ->
                        (WAVE_API_MAP[wave] ?: []).each { api ->
                            catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                ensureExchangeAsset(api)
                            }
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Create API Instance') {
            when { expression { !params.DRY_RUN } }
            steps {
                script {
                    WAVES_TO_RUN.each { wave ->
                        (WAVE_API_MAP[wave] ?: []).each { api ->
                            catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                def instanceId = findExistingInstance(api.assetId, api.instanceLabel)
                                if (instanceId) {
                                    log('INFO', "Reusing existing instance id=${instanceId} for label=${api.instanceLabel}")
                                    updateInstance(instanceId, api)
                                } else {
                                    instanceId = createInstance(api)
                                    log('INFO', "Created new instance id=${instanceId} for label=${api.instanceLabel}")
                                }
                                API_INSTANCES[api.instanceLabel] = instanceId
                            }
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Deploy API Instance') {
            when { expression { !params.DRY_RUN } }
            steps {
                script {
                    WAVES_TO_RUN.each { wave ->
                        (WAVE_API_MAP[wave] ?: []).each { api ->
                            def label      = api.instanceLabel
                            def instanceId = API_INSTANCES[label]
                            def status     = 'FAILED'
                            if (!instanceId) {
                                log('WARN', "No instance ID for ${api.appName} — Create API Instance likely failed, skipping deploy")
                                ALL_RESULTS << [wave: wave, label: label, status: 'FAILED', instanceId: '-']
                                return
                            }
                            catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                deployInstance(instanceId, api)
                                if (!params.SKIP_POLICIES && api.policies) {
                                    applyPolicies(instanceId, api.policies, api.appName)
                                }
                                validateInstance(instanceId, label)
                                status = 'OK'
                            }
                            ALL_RESULTS << [wave: wave, label: label, status: status, instanceId: instanceId]
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Summary') {
            steps {
                script {
                    echo '════════════════════ DEPLOYMENT SUMMARY ════════════════════'
                    (ALL_RESULTS ?: []).each { r ->
                        def icon = r.status == 'OK' ? 'OK      ' : (r.status == 'DRY_RUN' ? 'DRY_RUN ' : 'FAILED  ')
                        echo "  ${icon} | ${r.wave} | ${r.label.padRight(32)} | instanceId=${r.instanceId}"
                    }
                    echo '═════════════════════════════════════════════════════════════'
                    def failed = (ALL_RESULTS ?: []).findAll { it.status == 'FAILED' }
                    if (failed) {
                        // catchError marks this stage red and build FAILURE but lets
                        // the Postman Collection stage run afterwards.
                        catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                            error "${failed.size()} of ${(ALL_RESULTS ?: []).size()} deployment(s) failed: ${failed.collect { it.label }.join(', ')}"
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Postman Collection') {
            when {
                expression {
                    !params.DRY_RUN && (ALL_RESULTS ?: []).any { it.status == 'OK' }
                }
            }
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            input(
                                message: 'Generate Postman collection for local API testing?',
                                ok: 'Generate Postman Collection'
                            )
                        }
                        def outFile = generatePostmanCollection()
                        if (outFile) {
                            archiveArtifacts artifacts: outFile, allowEmptyArchive: false
                            log('INFO', 'Postman collection archived — download from Build Artifacts')
                        }
                    } catch (err) {
                        log('INFO', 'Postman collection skipped')
                    }
                }
            }
        }
    }

    post {
        success { echo "SUCCESS: all APIs deployed to ${env.FLEX_TARGET_NAME} (${params.ENVIRONMENT})" }
        failure { echo "FAILURE: one or more deployments failed. correlationId=${env.CORRELATION_ID}" }
        always  { script { env.ANYPOINT_TOKEN = '' } }
    }
}

// =============================================================================
// HELPER FUNCTIONS
// =============================================================================

// Return the Anypoint instance id if an API instance with the same assetId + label
// already exists, or null. Makes re-runs idempotent.
def findExistingInstance(String assetId, String label) {
    def response = apiCall('GET',
        "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis?assetId=${assetId}",
        null)
    def json  = readJSON text: response
    def match = json.assets?.collectMany { it.apis ?: [] }?.find { it.instanceLabel == label }
    return match ? "${match.id}" : null
}

// Patch an existing API instance's endpoint so proxyUri and upstreamUri
// stay in sync with the config repo on every pipeline run.
def updateInstance(String instanceId, Map api) {
    def body = writeJSON(returnText: true, json: [
        endpoint: [deploymentType: 'HY', uri: api.upstreamUri, proxyUri: api.proxyUri, isCloudHub: null]
    ])
    writeFile file: "update-${api.appName}.json", text: body
    try {
        apiCall('PATCH',
            "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}",
            "update-${api.appName}.json")
        log('INFO', "Updated instance ${instanceId} endpoint → proxyUri=${api.proxyUri}")
    } catch (err) {
        log('WARN', "Could not update instance ${instanceId} endpoint: ${err.message}")
    }
}

// Ensure the Exchange asset exists before creating an API Manager instance.
// GET /exchange/api/v2/assets/{group}/{assetId}/{version}:
//   200 → asset already there, nothing to do.
//   404 → publish a minimal HTTP API asset via multipart POST so createInstance can reference it.
def ensureExchangeAsset(Map api) {
    def groupId    = env.ORG_ID
    def assetId    = api.assetId
    def version    = api.assetVersion
    def apiVersion = "v${version.tokenize('.')[0]}"
    def name       = api.appName
    def checkUrl   = "${env.ANYPOINT_BASE_URL}/exchange/api/v2/assets/${groupId}/${assetId}/${version}"
    def publishUrl = "${env.ANYPOINT_BASE_URL}/exchange/api/v2/assets"

    def result = sh(
        script: """
            set -e
            TMP=\$(mktemp)
            HTTP=\$(curl -s -o "\$TMP" -w "%{http_code}" \\
                -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                -H "X-Correlation-ID: \$CORRELATION_ID" \\
                '${checkUrl}')
            if [ "\$HTTP" = "200" ]; then
                rm -f "\$TMP"; echo 'EXISTS'
            elif [ "\$HTTP" = "404" ]; then
                rm -f "\$TMP"
                TMP2=\$(mktemp)
                HTTP_POST=\$(curl -s -o "\$TMP2" -w "%{http_code}" \\
                    -X POST '${publishUrl}' \\
                    -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                    -H "X-Correlation-ID: \$CORRELATION_ID" \\
                    -F "organizationId=${groupId}" \\
                    -F "groupId=${groupId}" \\
                    -F "assetId=${assetId}" \\
                    -F "version=${version}" \\
                    -F "name=${name}" \\
                    -F "classifier=http" \\
                    -F "apiVersion=${apiVersion}")
                BODY=\$(cat "\$TMP2"); rm -f "\$TMP2"
                if [ "\$HTTP_POST" = "200" ] || [ "\$HTTP_POST" = "201" ]; then
                    echo 'PUBLISHED'
                elif [ "\$HTTP_POST" = "409" ]; then
                    if echo "\$BODY" | grep -q 'deleted'; then
                        echo 'DELETED_VERSION'
                    else
                        echo "ERROR:\${HTTP_POST}:\$BODY"
                    fi
                else
                    echo "ERROR:\${HTTP_POST}:\$BODY"
                fi
            else
                BODY=\$(cat "\$TMP"); rm -f "\$TMP"
                echo "ERROR:\${HTTP}:\$BODY"
            fi
        """,
        returnStdout: true
    ).trim()

    if (result == 'EXISTS') {
        log('INFO', "Exchange asset exists: ${assetId}:${version}")
    } else if (result == 'PUBLISHED') {
        log('INFO', "Published to Exchange as HTTP API: ${assetId}:${version}")
    } else if (result == 'DELETED_VERSION') {
        throw new Exception("Exchange asset ${assetId}:${version} was previously deleted — bump 'version' in apis/${api.appName}/config.yaml to a higher number and re-run.")
    } else {
        throw new Exception("Exchange asset check/publish failed for ${assetId}:${version}: ${result}")
    }
}

// Create a new API instance in API Manager. Returns the new instance id.
def createInstance(Map api) {
    def body = writeJSON(returnText: true, json: [
        spec         : [groupId: env.ORG_ID, assetId: api.assetId, version: api.assetVersion],
        endpoint     : [deploymentType: 'HY', uri: api.upstreamUri, proxyUri: api.proxyUri, isCloudHub: null],
        technology   : 'flexGateway',
        instanceLabel: api.instanceLabel
    ])
    writeFile file: "create-${api.appName}.json", text: body
    def response = apiCall('POST',
        "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis",
        "create-${api.appName}.json")
    def json = readJSON text: response
    if (!json.id) { error "createInstance returned no id for ${api.appName}. Response: ${response}" }
    return "${json.id}"
}

// Trigger deployment of an API instance to the Flex Gateway target.
// Strategy: GET first (Anypoint auto-creates a pending deployment on instance creation,
// so POST immediately returns 400 UniqueConstraintError). PATCH if id found; POST if not.
// Port-conflict 400s (InvalidOperationError "already in use") fail immediately with a
// clear actionable message — waiting won't help, it's a config problem.
def deployInstance(String instanceId, Map api) {
    def deployJson = [
        type          : 'HY',
        gatewayVersion: env.GATEWAY_VERSION,
        targetId      : env.FLEX_TARGET_ID,
        targetName    : env.FLEX_TARGET_NAME,
        environmentId : env.ENV_ID
    ]

    // Multi-endpoint routing: upstreams + routing are top-level fields in the
    // proxies xAPI deployment body, not nested inside overrides.
    def endpoints = (api?.endpoints ?: [])
    if (endpoints.size() > 1) {
        deployJson.upstreams = endpoints.collect { ep ->
            [label: ep.name, uri: ep.uri, weight: 100]
        }
        deployJson.routing = endpoints.collect { ep ->
            [
                label    : ep.name,
                upstreams: [[weight: 100, label: ep.name]],
                rules    : [path: "${ep.publicPath}(.*)"]
            ]
        }
    }

    def body = writeJSON(returnText: true, json: deployJson)
    def bodyFile = "deploy-${instanceId}.json"
    writeFile file: bodyFile, text: body
    log('INFO', "Deployment body for ${instanceId}: ${body}")

    sh """
        set -e
        DEPL_URL="\$ANYPOINT_BASE_URL/proxies/xapi/v1/organizations/\$ORG_ID/environments/\$ENV_ID/apis/${instanceId}/deployments"

        get_deploy_id() {
            curl -s \\
                -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                -H "X-Correlation-ID: \$CORRELATION_ID" \\
                "\$DEPL_URL" | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    if isinstance(d, list): d = d[0] if d else {}
    print(d.get('id', '') if isinstance(d, dict) else '')
except: print('')
" 2>/dev/null || true
        }

        DEPL_ID=\$(get_deploy_id)
        if [ -z "\$DEPL_ID" ] || [ "\$DEPL_ID" = "null" ]; then
            sleep 5
            DEPL_ID=\$(get_deploy_id)
        fi

        if [ -n "\$DEPL_ID" ] && [ "\$DEPL_ID" != "null" ]; then
            curl -s -X PATCH "\$DEPL_URL/\$DEPL_ID" \\
                -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                -H "X-Correlation-ID: \$CORRELATION_ID" \\
                -H "Content-Type: application/json" \\
                --data "@${bodyFile}" > /dev/null
            echo "Deployment updated (id=\$DEPL_ID) for instance ${instanceId}"
            echo "[DEBUG] Stored deployment (GET):"
            curl -s "\$DEPL_URL/\$DEPL_ID" \\
                -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                -H "X-Correlation-ID: \$CORRELATION_ID"
        else
            TMP=\$(mktemp)
            HTTP=\$(curl -s -o "\$TMP" -w "%{http_code}" -X POST "\$DEPL_URL" \\
                -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                -H "X-Correlation-ID: \$CORRELATION_ID" \\
                -H "Content-Type: application/json" \\
                --data "@${bodyFile}")
            BODY=\$(cat "\$TMP"); rm -f "\$TMP"
            if [ "\$HTTP" = "200" ] || [ "\$HTTP" = "201" ]; then
                echo "Deployment created for instance ${instanceId}"
                NEW_ID=\$(echo "\$BODY" | python3 -c "import sys,json; print(json.load(sys.stdin).get('id',''))" 2>/dev/null || true)
                if [ -n "\$NEW_ID" ]; then
                    echo "[DEBUG] Stored deployment (GET):"
                    curl -s "\$DEPL_URL/\$NEW_ID" \\
                        -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                        -H "X-Correlation-ID: \$CORRELATION_ID"
                fi
            elif [ "\$HTTP" = "400" ]; then
                if echo "\$BODY" | grep -q 'already in use\\|InvalidOperationError'; then
                    echo "PORT CONFLICT on Flex Target ${env.FLEX_TARGET_NAME}: \$BODY\\nFix: assign a different proxyUri in envs/${params.ENVIRONMENT}/overlay.yaml, or undeploy the conflicting API first." >&2
                    exit 1
                fi
                sleep 10
                DEPL_ID=\$(get_deploy_id)
                if [ -n "\$DEPL_ID" ] && [ "\$DEPL_ID" != "null" ]; then
                    curl -s -X PATCH "\$DEPL_URL/\$DEPL_ID" \\
                        -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                        -H "X-Correlation-ID: \$CORRELATION_ID" \\
                        -H "Content-Type: application/json" \\
                        --data "@${bodyFile}" > /dev/null
                    echo "Deployment updated after retry (id=\$DEPL_ID) for instance ${instanceId}"
                else
                    echo "Deploy failed HTTP \$HTTP: \$BODY — deployment id unavailable after 10s retry" >&2
                    exit 1
                fi
            else
                echo "Deploy failed HTTP \$HTTP: \$BODY" >&2
                exit 1
            fi
        fi
    """
}

// Apply policies from config, skipping any already applied (idempotent).
def applyPolicies(String instanceId, List policies, String appName) {
    // Fetch already-applied policies — wrap in try/catch so a GET failure
    // (e.g. 404 on a brand-new instance) doesn't abort the whole deployment.
    List policiesList = []
    try {
        def existingResp = apiCall('GET',
            "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}/policies",
            null)
        log('INFO', "GET policies response (first 500 chars): ${existingResp.take(500)}")
        def respData = readJSON text: existingResp
        // Anypoint may return a bare [] or a wrapped {"policies":[...]} object.
        if (respData instanceof List) {
            policiesList = respData as List
        } else if (respData?.policies instanceof List) {
            policiesList = respData.policies as List
        }
    } catch (err) {
        log('WARN', "Could not fetch existing policies for ${appName}: ${err.message} — will attempt all policies")
    }
    // Anypoint stores Flex Gateway policy variants with a '-flex' suffix (e.g. 'client-id-enforcement-flex')
    // but config.yaml uses the base name ('client-id-enforcement'). Strip the suffix so matching works.
    def existingIds = policiesList.collect {
        def raw = it.template?.assetId ?: it.implementationAsset?.assetId ?: it.assetId ?: ''
        raw.endsWith('-flex') ? raw.replace('-flex', '') : raw
    }.findAll { it }

    policies.each { policy ->
        if (policy.assetId in existingIds) {
            log('INFO', "Policy '${policy.assetId}' already on ${appName} — skipping")
            return
        }
        def body = writeJSON(returnText: true, json: [
            configurationData: policy.config,
            pointcutData     : null,
            groupId          : policy.groupId,
            assetId          : policy.assetId,
            assetVersion     : policy.policyVersion
        ])
        def policyFile = "policy-${instanceId}-${policy.assetId}.json"
        writeFile file: policyFile, text: body
        def policyUrl  = "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}/policies"
        def policyResp = sh(
            script: """
                set -e
                TMP=\$(mktemp)
                HTTP=\$(curl -s -o "\$TMP" -w "%{http_code}" \\
                    -X POST '${policyUrl}' \\
                    -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                    -H "X-Correlation-ID: \$CORRELATION_ID" \\
                    -H "Content-Type: application/json" \\
                    --data "@${policyFile}")
                BODY=\$(cat "\$TMP"); rm -f "\$TMP"
                if [ "\$HTTP" = "200" ] || [ "\$HTTP" = "201" ]; then
                    echo 'APPLIED'
                elif [ "\$HTTP" = "409" ]; then
                    echo 'DUPLICATE'
                else
                    echo "ERROR:\${HTTP}:\$BODY"
                fi
            """,
            returnStdout: true
        ).trim()
        if (policyResp == 'APPLIED') {
            log('INFO', "Applied policy '${policy.assetId}' to ${appName}")
        } else if (policyResp == 'DUPLICATE') {
            log('INFO', "Policy '${policy.assetId}' already applied to ${appName} — skipping (HTTP 409)")
        } else {
            log('WARN', "Policy '${policy.assetId}' failed for ${appName}: ${policyResp}")
        }
    }
}

// Poll Anypoint proxies xAPI until the Flex Gateway deployment reaches a terminal state.
// API Manager's deployment.applicationStatus is always null for Flex Gateway (HY)
// instances — the real status lives in the proxies xAPI deployment object.
// If the status field is absent (UNKNOWN) for 3 consecutive checks, we give up and
// log WARN so the build doesn't stall for 5 minutes on an unresolvable field.
def validateInstance(String instanceId, String label) {
    def deplUrl       = "${env.ANYPOINT_BASE_URL}/proxies/xapi/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}/deployments"
    def maxChecks     = 30  // 5 minutes (30 × 10 s)
    def unknownStreak = 0
    for (int i = 0; i < maxChecks; i++) {
        sleep(time: 10, unit: 'SECONDS')
        def response = apiCall('GET', deplUrl, null)
        // Log the raw response on the first poll to reveal the actual field structure
        if (i == 0) {
            log('INFO', "${label}  raw deployment response (first 500 chars): ${response.take(500)}")
        }
        def respData = readJSON text: response
        // Resolve deployment object — handles bare object, array, items-wrapper, and deployments-wrapper
        def depl = [:]
        if (respData instanceof List) {
            depl = respData ? respData[0] : [:]
        } else if (respData?.items instanceof List) {
            def items = respData.items as List
            depl = items ? items[0] : [:]
        } else if (respData?.deployments instanceof List) {
            def depls = respData.deployments as List
            depl = depls ? depls[0] : [:]
        } else {
            depl = respData ?: [:]
        }
        // Try the most common status field names across Anypoint xAPI versions
        def status = (depl.status ?: depl.applicationStatus ?: depl.deploymentStatus ?: 'UNKNOWN').toUpperCase()
        def pct    = (int)((i + 1) * 100 / maxChecks)
        def bar    = ('#' * (int)(pct / 5)).padRight(20, '-')
        log('INFO', "${label}  [${bar}] ${pct}%  check ${i + 1}/${maxChecks}: gatewayStatus=${status}")
        if (status in ['DEPLOYED', 'STARTED', 'ACTIVE', 'APPLIED']) {
            log('INFO', "${label}  confirmed active on gateway (status=${status})")
            return
        }
        if (status == 'FAILED') {
            // Flex GW can briefly report FAILED after a PATCH while still serving the prior config
            log('WARN', "${label}  gateway status=FAILED — API may still serve traffic; verify in Anypoint Runtime Manager")
            return
        }
        if (status == 'UNKNOWN') {
            unknownStreak++
            if (unknownStreak >= 3) {
                log('WARN', "${label}  status field absent in proxies xAPI response for ${unknownStreak} consecutive checks — verify in Anypoint Runtime Manager; proceeding")
                return
            }
        } else {
            unknownStreak = 0
        }
    }
    log('WARN', "${label}  did not confirm DEPLOYED within ${(int)(maxChecks * 10 / 60)} min — check Anypoint Runtime Manager; proceeding")
}

// Authenticated Anypoint API call using curl + python3 (Linux-compatible).
// Token is never logged — masked by withCredentials in the Authenticate stage.
def apiCall(String method, String url, String bodyFile) {
    def bodyArg = bodyFile ? "--data @\"${bodyFile}\"" : ''
    def raw = sh(
        script: """
            set -e
            TMP=\$(mktemp)
            HTTP=\$(curl -s -o "\$TMP" -w "%{http_code}" \\
                -X '${method}' '${url}' \\
                -H "Authorization: Bearer \$ANYPOINT_TOKEN" \\
                -H "X-Correlation-ID: \$CORRELATION_ID" \\
                -H "Content-Type: application/json" \\
                ${bodyArg})
            BODY=\$(cat "\$TMP"); rm -f "\$TMP"
            if [ "\$HTTP" -lt 200 ] || [ "\$HTTP" -ge 300 ]; then
                echo "API call failed [${method} ${url}] HTTP \$HTTP: \$BODY" >&2
                exit 1
            fi
            printf '%s' "\$BODY"
        """,
        returnStdout: true
    ).trim()
    return raw
}

// Build a Postman Collection v2.1 JSON for each successfully deployed API.
// Reads publicPath from apis/<app>/config.yaml and publicHostname from the env overlay.
// Returns the output filename (for archiveArtifacts), or null if nothing to write.
def generatePostmanCollection() {
    def deployedResults = (ALL_RESULTS ?: []).findAll { it.status == 'OK' }
    if (!deployedResults) {
        log('WARN', 'No successfully deployed APIs — skipping Postman collection')
        return null
    }

    // Public hostname from overlay defaults (falls back to localhost)
    def gatewayHost = 'localhost'
    def overlayPath = "envs/${params.ENVIRONMENT}/overlay.yaml"
    if (fileExists(overlayPath)) {
        def ov = readYaml file: overlayPath
        gatewayHost = "${ov.defaults?.publicHostname ?: 'localhost'}"
    }

    def folders = []

    deployedResults.each { result ->
        def api = (WAVE_API_MAP[result.wave] ?: []).find { it.instanceLabel == result.label }
        if (!api) return

        // Extract port from proxyUri: http://0.0.0.0:8082 → 8082
        def proxyPort = api.proxyUri.tokenize(':').last().replaceAll('[^0-9]', '') ?: '8080'

        // Re-read config.yaml to get the endpoint publicPaths
        def apiCfg    = readYaml file: "apis/${api.appName}/config.yaml"
        def endpoints = apiCfg.endpoints ?: []

        // Headers driven by applied policies
        def headers = [[key: 'Content-Type', value: 'application/json', type: 'text']]
        if (api.policies?.any { it.assetId == 'client-id-enforcement' }) {
            headers << [key: 'client_id',    value: '{{client_id}}',    type: 'text']
            headers << [key: 'client_secret', value: '{{client_secret}}', type: 'text']
        }

        def requests = endpoints.collect { ep ->
            def rawPath  = (ep.publicPath ?: '/').replaceAll('^/', '')
            def segments = rawPath ? rawPath.split('/').toList() : []
            [
                name    : "${ep.name ?: api.appName}",
                request : [
                    method : 'GET',
                    header : headers,
                    url    : [
                        raw     : "http://{{gateway_host}}:${proxyPort}/${rawPath}",
                        protocol: 'http',
                        host    : ['{{gateway_host}}'],
                        port    : "${proxyPort}",
                        path    : segments
                    ]
                ],
                response: []
            ]
        }

        folders << [
            name: "${api.appName}  [${result.wave}]",
            item: requests
        ]
    }

    def collection = [
        info    : [
            name  : "Flex GW — ${params.ENVIRONMENT} / ${params.WAVE}",
            schema: 'https://schema.getpostman.com/json/collection/v2.1.0/collection.json'
        ],
        variable: [
            [key: 'gateway_host',  value: "${gatewayHost}",       type: 'string'],
            [key: 'client_id',     value: 'YOUR_CLIENT_ID',       type: 'string'],
            [key: 'client_secret', value: 'YOUR_CLIENT_SECRET',   type: 'string']
        ],
        item    : folders
    ]

    def outFile = "postman-collection-${params.ENVIRONMENT}-${params.WAVE}-${env.BUILD_NUMBER}.json"
    writeJSON file: outFile, json: collection, pretty: 4
    log('INFO', "Postman collection written → ${outFile}")
    return outFile
}

// JSON-structured, correlation-aware log line.
def log(String level, String message) {
    echo "{\"level\":\"${level}\",\"correlationId\":\"${env.CORRELATION_ID}\",\"stage\":\"${env.STAGE_NAME}\",\"msg\":\"${message}\"}"
}
