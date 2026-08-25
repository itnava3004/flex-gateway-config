// =============================================================================
// Jenkinsfile — Wave-based deploy to Anypoint Flex Gateway
//               Driven by flex-gateway-config repo (GitOps)
// -----------------------------------------------------------------------------
// Flow:
//   1. Checkout flex-gateway-config repo (or the repo this Jenkinsfile lives in)
//   2. Read anypoint.yaml → resolve orgId, envId, flexTarget for chosen ENVIRONMENT
//   3. Read waves/<WAVE>/manifest.yaml → list of apps to deploy
//   4. For each app: read apis/<app>/config.yaml + envs/<app>/<env>/runtime.yaml
//   5. Approval gate for test/qa/preprod/prod (skipped for dev/sandbox + DRY_RUN)
//   6. Authenticate via Connected App (client_credentials)
//   7. Deploy each wave in order:
//        find-or-create instance → deploy → apply policies → validate
//   8. Print summary; fail build if any API failed
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

        string(name: 'PREPROD_PROD_APPROVERS',
               defaultValue: '3672738',
               description: 'Approvers allowed to deploy to PREPROD and PROD envs (comma-separated Jenkins users/groups)')
        string(name: 'TEST_QA_APPROVERS',
               defaultValue: '3672738',
               description: 'Approvers allowed to deploy to TEST & QA envs (comma-separated Jenkins users/groups)')
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
        // Must exceed the 48h approval window below, otherwise the build is
        // aborted while still waiting for a higher-environment approver. Hang
        // protection for the actual work now lives on the individual network
        // stages (30m each) rather than on the pipeline as a whole.
        timeout(time: 49, unit: 'HOURS')
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

                        def apiList = []
                        apps.each { appName ->
                            def cfgPath = "apis/${appName}/config.yaml"
                            if (!fileExists(cfgPath)) {
                                error "API config not found: ${cfgPath}"
                            }
                            def apiCfg = readYaml file: cfgPath

                            // Per-app, per-environment runtime config: backend URIs, proxy
                            // port and gateway hostname. Try the environment name as given,
                            // then lower/upper case, so 'dev' and 'DEV' both resolve on a
                            // case-sensitive agent filesystem.
                            def rtPath = ["envs/${appName}/${params.ENVIRONMENT}/runtime.yaml",
                                          "envs/${appName}/${params.ENVIRONMENT.toLowerCase()}/runtime.yaml",
                                          "envs/${appName}/${params.ENVIRONMENT.toUpperCase()}/runtime.yaml"]
                                         .find { fileExists(it) }
                            if (!rtPath) {
                                error "Runtime config not found for ${appName} in '${params.ENVIRONMENT}' — expected envs/${appName}/${params.ENVIRONMENT}/runtime.yaml"
                            }
                            def runtime = readYaml file: rtPath

                            // Flatten the endpoint→URI map into a plain Groovy Map so that
                            // hyphenated keys (e.g. customer-address) are safe to look up
                            // inside closures under Jenkins CPS serialisation.
                            def epUris = [:]
                            runtime.endpoints?.each { k, v -> epUris["${k}"] = "${v}" }

                            // Join the env-independent contract (config.yaml) with the
                            // per-env backend URIs (runtime.yaml), keyed on endpoint name.
                            def endpoints = (apiCfg.endpoints ?: []).collect { ep ->
                                def epName = "${ep.name}"
                                def uri    = epUris[epName]
                                if (!uri) {
                                    error "No URI for endpoint '${epName}' of ${appName} in ${rtPath} — add it under endpoints:"
                                }
                                [name: epName, uri: uri, publicPath: "${ep.publicPath ?: ''}"]
                            }
                            if (!endpoints) {
                                error "No endpoints declared in ${cfgPath}"
                            }
                            def orphans = epUris.keySet() - endpoints.collect { it.name }
                            if (orphans) {
                                log('WARN', "${appName}: ${rtPath} defines URIs for unknown endpoints ${orphans.join(', ')} — not present in ${cfgPath}")
                            }

                            // upstreamUri: explicit override in runtime.yaml, else first endpoint
                            def upstreamUri = "${runtime.upstreamUri ?: endpoints[0].uri}"
                            // proxyUri: from runtime.yaml, else a sane default on port 8080
                            def proxyUri    = "${runtime.proxyUri ?: 'http://0.0.0.0:8080'}"
                            log('INFO', "  ${appName}: proxyUri=${proxyUri}  upstream=${upstreamUri}  (${endpoints.size()} endpoints from ${rtPath})")

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
                                appName          : "${appName}",
                                assetId          : "${apiCfg.assetId}",
                                assetVersion     : "${apiCfg.version}",
                                // Exchange display name; folder name if not declared
                                exchangeAssetName: "${apiCfg.exchangeAssetName ?: appName}",
                                instanceLabel: "${appName}-${params.ENVIRONMENT}",
                                upstreamUri  : upstreamUri,
                                proxyUri     : proxyUri,
                                policies     : policies,
                                wave         : wave,
                                endpoints    : endpoints
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
        // Human sign-off for anything above dev/sandbox. Deliberately placed
        // BEFORE Authenticate: the OAuth token is short-lived, so acquiring it
        // first and then waiting up to 48h for an approver would leave the rest
        // of the pipeline holding an expired token. Nothing in Anypoint has been
        // touched at this point, so a rejection is a clean no-op.
        stage('Approval: Higher Environment Gate') {
            when {
                expression {
                    !params.DRY_RUN && !(params.ENVIRONMENT.toLowerCase() in ['dev', 'sandbox'])
                }
            }
            steps {
                script {
                    def envKey        = params.ENVIRONMENT.toLowerCase()
                    def approversList = (envKey in ['test', 'qa'])
                        ? params.TEST_QA_APPROVERS
                        : params.PREPROD_PROD_APPROVERS

                    if (!approversList?.trim()) {
                        error "Approver list for ${params.ENVIRONMENT} is empty. Cannot request approval."
                    }

                    def apiNames = (WAVES_TO_RUN ?: []).collectMany { w ->
                        (WAVE_API_MAP[w] ?: []).collect { "${it.appName}" }
                    }
                    log('INFO', "Awaiting approval to deploy [${apiNames.join(', ')}] to ${params.ENVIRONMENT} (approvers: ${approversList})")

                    def approval = timeout(time: 48, unit: 'HOURS') {
                        input(
                            id: 'HigherEnvApproval',
                            message: "Approve deployment of ${params.WAVE} [${apiNames.join(', ')}] to ${params.ENVIRONMENT}?",
                            ok: 'Approve & Continue',
                            submitter: approversList,
                            submitterParameter: 'APPROVED_BY',
                            parameters: [
                                text(name: 'JUSTIFICATION',
                                     defaultValue: '',
                                     description: 'Reason / CHG / Ticket for this deployment (required)')
                            ]
                        )
                    }

                    env.APPROVED_BY            = approval['APPROVED_BY'] ?: 'unknown'
                    env.APPROVAL_JUSTIFICATION = approval['JUSTIFICATION'] ?: '(none)'

                    log('INFO', "${params.ENVIRONMENT} deployment approved by: ${env.APPROVED_BY}")
                    log('INFO', "Justification: ${env.APPROVAL_JUSTIFICATION}")

                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${params.WAVE} -> ${params.ENVIRONMENT}"
                    currentBuild.description = "Approved by ${env.APPROVED_BY} | ${env.APPROVAL_JUSTIFICATION}"
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Authenticate to Anypoint') {
            when { expression { !params.DRY_RUN } }
            options { timeout(time: 30, unit: 'MINUTES') }
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
            options { timeout(time: 30, unit: 'MINUTES') }
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
            options { timeout(time: 30, unit: 'MINUTES') }
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
                                configureRouting(instanceId, api)
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
            options { timeout(time: 30, unit: 'MINUTES') }
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
                        error "${failed.size()} of ${(ALL_RESULTS ?: []).size()} deployment(s) failed: ${failed.collect { it.label }.join(', ')}"
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
    def updateJson = [
        endpoint: [deploymentType: 'HY', uri: api.upstreamUri, proxyUri: api.proxyUri, isCloudHub: null]
    ]
    // Routing is handled separately by configureRouting() — see note in createInstance().
    def body = writeJSON(returnText: true, json: updateJson)
    writeFile file: "update-${api.appName}.json", text: body
    try {
        def resp = apiCall('PATCH',
            "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}",
            "update-${api.appName}.json")
        log('INFO', "Updated instance ${instanceId} endpoint → proxyUri=${api.proxyUri}. Response: ${resp.take(500)}")
    } catch (err) {
        log('WARN', "Could not update instance ${instanceId} endpoint: ${err.message}")
    }
}

// Configure path-based routing for APIs with multiple endpoints (Option B:
// one instance, one port, per-path upstreams).
//
// Anypoint drops `routing` when supplied in the create/update body — upstreams are
// separate resources with server-assigned UUIDs, and routing entries must reference
// those ids. So: create the upstreams first, then PATCH routing with their ids.
def configureRouting(String instanceId, Map api) {
    def endpoints = (api?.endpoints ?: [])
    if (endpoints.size() < 2) { return }

    def base = "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}"
    def uriToId = [:]

    // ── Existing upstreams (idempotent re-runs) ──
    try {
        def existing = apiCall('GET', "${base}/upstreams", null)
        log('INFO', "GET upstreams for ${api.appName}: ${existing.take(1000)}")
        def parsed = readJSON text: existing
        def list   = (parsed instanceof List) ? parsed : (parsed?.upstreams ?: [])
        list.each { u -> if (u?.uri && u?.id) { uriToId["${u.uri}"] = "${u.id}" } }
    } catch (err) {
        log('WARN', "Could not GET upstreams for ${api.appName}: ${err.message}")
    }

    // ── Create any upstream that doesn't exist yet ──
    endpoints.each { ep ->
        if (uriToId["${ep.uri}"]) { return }
        def upFile = "upstream-${api.appName}-${ep.name}.json"
        writeFile file: upFile, text: writeJSON(returnText: true, json: [label: ep.name, uri: ep.uri])
        try {
            def resp = apiCall('POST', "${base}/upstreams", upFile)
            log('INFO', "POST upstream '${ep.name}' for ${api.appName}: ${resp.take(500)}")
            def created = readJSON text: resp
            if (created?.id) { uriToId["${ep.uri}"] = "${created.id}" }
        } catch (err) {
            log('WARN', "Could not create upstream '${ep.name}' for ${api.appName}: ${err.message}")
        }
    }

    // ── PATCH routing, referencing upstreams by their server-assigned ids ──
    // label comes from the endpoint's `name` in config.yaml so routes are
    // identifiable in the Anypoint UI rather than showing as unnamed.
    def routes = endpoints.findAll { uriToId["${it.uri}"] }.collect { ep ->
        [
            label    : "${ep.name}",
            upstreams: [[id: uriToId["${ep.uri}"], weight: 100]],
            rules    : [path: "${ep.publicPath}(.*)"]
        ]
    }
    if (routes.size() < endpoints.size()) {
        log('WARN', "Only ${routes.size()}/${endpoints.size()} upstreams resolved for ${api.appName} — routing will be incomplete")
    }
    if (!routes) {
        log('WARN', "No upstreams resolved for ${api.appName} — skipping routing PATCH")
        return
    }

    def routeFile = "routing-${api.appName}.json"
    def routeBody = writeJSON(returnText: true, json: [routing: routes])
    log('INFO', "PATCH routing body for ${api.appName}: ${routeBody}")
    writeFile file: routeFile, text: routeBody
    try {
        def resp = apiCall('PATCH', base, routeFile)
        log('INFO', "PATCH routing response for ${api.appName}: ${resp.take(1000)}")
        log('INFO', "Configured ${routes.size()} routes for ${api.appName}: ${endpoints.collect { it.publicPath }.join(', ')}")
    } catch (err) {
        log('WARN', "Could not PATCH routing for ${api.appName}: ${err.message}")
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
    // Exchange display name: exchangeAssetName from config.yaml, falling back to
    // the folder name so configs predating that field keep their current name.
    def name       = api.exchangeAssetName
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
        log('INFO', "Published to Exchange as HTTP API: ${assetId}:${version} (name='${name}')")
    } else if (result == 'DELETED_VERSION') {
        throw new Exception("Exchange asset ${assetId}:${version} was previously deleted — bump 'version' in apis/${api.appName}/config.yaml to a higher number and re-run.")
    } else {
        throw new Exception("Exchange asset check/publish failed for ${assetId}:${version}: ${result}")
    }
}

// Create a new API instance in API Manager. Returns the new instance id.
def createInstance(Map api) {
    def createJson = [
        spec         : [groupId: env.ORG_ID, assetId: api.assetId, version: api.assetVersion],
        endpoint     : [deploymentType: 'HY', uri: api.upstreamUri, proxyUri: api.proxyUri, isCloudHub: null],
        technology   : 'flexGateway',
        instanceLabel: api.instanceLabel
    ]
    // NOTE: routing is NOT accepted in the create body — Anypoint silently drops it.
    // Upstreams are separate resources with server-assigned UUIDs; see configureRouting().
    def body = writeJSON(returnText: true, json: createJson)
    writeFile file: "create-${api.appName}.json", text: body
    def response = apiCall('POST',
        "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis",
        "create-${api.appName}.json")
    log('INFO', "createInstance response for ${api.appName}: ${response}")
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

    // NOTE: routing is managed via the API Manager API (createInstance/updateInstance),
    // not the proxies xAPI deployment body — Anypoint ignores routing fields here for HY type.
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
                    echo "PORT CONFLICT on Flex Target ${env.FLEX_TARGET_NAME}: \$BODY\\nFix: assign a different proxyUri in envs/${api.appName}/${params.ENVIRONMENT}/runtime.yaml, or undeploy the conflicting API first." >&2
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

// JSON-structured, correlation-aware log line.
def log(String level, String message) {
    echo "{\"level\":\"${level}\",\"correlationId\":\"${env.CORRELATION_ID}\",\"stage\":\"${env.STAGE_NAME}\",\"msg\":\"${message}\"}"
}
