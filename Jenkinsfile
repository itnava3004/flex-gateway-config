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
// Required agent tools     : git, curl, jq
// Required credentials     : anypoint-connected-app-<env>
//                             (Kind: Username+Password, user=CLIENT_ID, pass=CLIENT_SECRET)
// =============================================================================

pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT',
               choices: ['dev', 'test', 'qa', 'preprod', 'prod','sandbox'],
               description: 'Target Anypoint environment')
        choice(name: 'WAVE',
               choices: ['all', 'R1', 'R2', 'R3', 'R4'],
               description: '"all" runs R1 → R2 → … in sequence; pick a specific wave for targeted deploy')
        string(name: 'APP_FILTER',
               defaultValue: '',
               description: 'Optional: comma/space-separated app names to limit within the wave (e.g. "payments-api,orders-api"). Blank = all apps in the wave.')
        string(name: 'CONFIG_REPO_URL',
               defaultValue: '',
               description: 'Git URL of flex-gateway-config repo. Leave blank to use the repo this Jenkinsfile is checked in to.')
        string(name: 'CONFIG_BRANCH',
               defaultValue: 'main',
               description: 'Branch of the config repo')
        string(name: 'CONFIG_CRED_ID',
               defaultValue: '',
               description: 'Jenkins credentials ID for Git auth on the config repo (blank = public/anonymous)')
        booleanParam(name: 'DRY_RUN',
               defaultValue: true,
               description: 'Print the deployment plan without calling Anypoint APIs. Default true — set false to actually deploy.')
        booleanParam(name: 'SKIP_POLICIES',
               defaultValue: false,
               description: 'Skip policy application step (useful for initial connectivity testing)')
    }

    environment {
        CORRELATION_ID        = "${env.BUILD_TAG}"
        // MuleSoft's Exchange group ID — applies to all built-in managed policies
        MULESOFT_POLICY_GROUP = '68ef9520-24e9-4cf2-b2f5-620025690913'
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
                    if (params.CONFIG_REPO_URL?.trim()) {
                        log('INFO', "Cloning config repo: ${params.CONFIG_REPO_URL} @ ${params.CONFIG_BRANCH}")
                        if (params.CONFIG_CRED_ID?.trim()) {
                            git url: params.CONFIG_REPO_URL,
                                branch: params.CONFIG_BRANCH,
                                credentialsId: params.CONFIG_CRED_ID
                        } else {
                            git url: params.CONFIG_REPO_URL, branch: params.CONFIG_BRANCH
                        }
                    } else {
                        log('INFO', 'Using the repo this Jenkinsfile is in (checkout scm)')
                        checkout scm
                    }
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
                    def envBlock = acfg.anypoint?.environments?.get(params.ENVIRONMENT)
                    if (!envBlock) {
                        error "No anypoint config found for environment '${params.ENVIRONMENT}' in anypoint.yaml"
                    }
                    env.ORG_ID            = "${acfg.anypoint.organizationId}"
                    env.ENV_ID            = "${envBlock.environmentId}"
                    env.GATEWAY_VERSION   = "${envBlock.gatewayVersion}"
                    env.FLEX_TARGET_ID    = "${envBlock.flexTarget.id}"
                    env.FLEX_TARGET_NAME  = "${envBlock.flexTarget.name}"
                    env.ANYPOINT_BASE_URL = "${acfg.anypoint.baseUrl ?: 'https://anypoint.mulesoft.com'}"

                    log('INFO', "Target: env=${params.ENVIRONMENT} flexTarget=${env.FLEX_TARGET_NAME}")

                    // ── Determine waves ────────────────────────────────────────
                    def wavesToRun = params.WAVE == 'all'
                        ? ['R1', 'R2', 'R3', 'R4'].findAll { fileExists("waves/${it}/manifest.yaml") }
                        : [params.WAVE]
                    log('INFO', "Waves to run: ${wavesToRun.join(', ')}")

                    // ── Optional app-level filter ──────────────────────────────
                    def appFilter = params.APP_FILTER?.trim()
                        ? params.APP_FILTER.split('[,\\s]+').collect { it.trim() }.findAll { it }
                        : []

                    // ── Build wave → api-list map ──────────────────────────────
                    // Policy asset-id normalisation: some configs use the short
                    // name; Anypoint Exchange requires the full asset ID.
                    def POLICY_ID_NORM = ['rate-limiting': 'rate-limiting-sla']

                    def waveApiMap = [:]
                    wavesToRun.each { wave ->
                        def manifestPath = "waves/${wave}/manifest.yaml"
                        if (!fileExists(manifestPath)) {
                            log('WARN', "Wave ${wave}: manifest not found at ${manifestPath} — skipping")
                            return
                        }
                        def manifest = readYaml file: manifestPath
                        def apps = (manifest.apps ?: []).collect { "${it}" }

                        if (appFilter) {
                            apps = apps.findAll { appFilter.contains(it) }
                            log('INFO', "Wave ${wave}: filtered to [${apps.join(', ')}]")
                        }

                        def overlayPath = "envs/${params.ENVIRONMENT}/overlay.yaml"
                        def overlay     = fileExists(overlayPath) ? readYaml(file: overlayPath) : [:]

                        def apiList = []
                        apps.each { appName ->
                            def cfgPath = "apis/${appName}/config.yaml"
                            if (!fileExists(cfgPath)) {
                                error "API config not found: ${cfgPath}"
                            }
                            def apiCfg     = readYaml file: cfgPath
                            def appOverlay = overlay?.apps?.get(appName) ?: [:]

                            // upstreamUri: env overlay wins, then first endpoint's backend URI
                            def upstreamUri = "${appOverlay.upstreamUri ?: apiCfg.endpoints[0].uri}"
                            // proxyUri: env overlay wins, then a sane default on port 8080
                            def proxyUri    = "${appOverlay.proxyUri    ?: 'http://0.0.0.0:8080'}"

                            def policies = (apiCfg.policies ?: []).collect { p ->
                                [
                                    assetId      : "${POLICY_ID_NORM[p.assetId] ?: p.assetId}",
                                    groupId      : "${p.groupId ?: env.MULESOFT_POLICY_GROUP}",
                                    policyVersion: "${p.policyVersion}",
                                    config       : p.config ?: [:]
                                ]
                            }

                            apiList << [
                                appName      : "${appName}",
                                assetId      : "${apiCfg.assetId}",
                                assetVersion : "${apiCfg.version}",
                                instanceLabel: "${appName}-${params.ENVIRONMENT}",
                                upstreamUri  : upstreamUri,
                                proxyUri     : proxyUri,
                                policies     : policies,
                                wave         : wave
                            ]
                        }
                        waveApiMap[wave] = apiList
                        log('INFO', "Wave ${wave}: [${apiList.collect { it.appName }.join(', ')}]")
                    }

                    WAVES_TO_RUN = wavesToRun
                    WAVE_API_MAP = waveApiMap
                }
            }
        }

        // ──────────────────────────────────────────────────────────────────── //
        stage('Authenticate') {
            when { expression { !params.DRY_RUN } }
            steps {
                withCredentials([usernamePassword(
                        credentialsId: "anypoint-connected-app-${params.ENVIRONMENT}",
                        usernameVariable: 'CLIENT_ID',
                        passwordVariable: 'CLIENT_SECRET')]) {
                    script {
                        log('INFO', 'Requesting OAuth2 access token via client_credentials grant')
                        def token = powershell(
                            script: '''
                                $ErrorActionPreference = "Stop"
                                $body = '{"grant_type":"client_credentials","client_id":"' + $env:CLIENT_ID + '","client_secret":"' + $env:CLIENT_SECRET + '"}'
                                $resp = Invoke-WebRequest -Method POST `
                                    -Uri "$env:ANYPOINT_BASE_URL/accounts/api/v2/oauth2/token" `
                                    -ContentType "application/json" `
                                    -Body $body `
                                    -UseBasicParsing
                                ($resp.Content | ConvertFrom-Json).access_token
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
        stage('Deploy Waves') {
            steps {
                script {
                    def allResults = []

                    WAVES_TO_RUN.each { wave ->
                        def apis = WAVE_API_MAP[wave] ?: []
                        if (!apis) {
                            log('INFO', "Wave ${wave} has no APIs after filtering — skipping")
                            return
                        }
                        echo "─────────────────────────── WAVE ${wave} ───────────────────────────"

                        apis.each { api ->
                            def label = api.instanceLabel
                            try {
                                if (params.DRY_RUN) {
                                    log('INFO', "[DRY_RUN] wave=${wave}  app=${api.appName}  ${api.assetId}:${api.assetVersion}")
                                    log('INFO', "[DRY_RUN]   upstream=${api.upstreamUri}  proxy=${api.proxyUri}")
                                    log('INFO', "[DRY_RUN]   policies=${api.policies?.collect { it.assetId }}")
                                    allResults << [wave: wave, label: label, status: 'DRY_RUN', instanceId: '-']
                                    return
                                }

                                // Idempotent: re-running the pipeline updates instead of duplicating
                                def instanceId = findExistingInstance(api.assetId, label)
                                if (instanceId) {
                                    log('INFO', "Reusing existing instance id=${instanceId} for label=${label}")
                                } else {
                                    instanceId = createInstance(api)
                                    log('INFO', "Created new instance id=${instanceId} for label=${label}")
                                }

                                deployInstance(instanceId)

                                if (!params.SKIP_POLICIES && api.policies) {
                                    applyPolicies(instanceId, api.policies, api.appName)
                                }

                                validateInstance(instanceId, label)
                                allResults << [wave: wave, label: label, status: 'OK', instanceId: instanceId]

                            } catch (err) {
                                log('ERROR', "wave=${wave}  label=${label}  error=${err.message}")
                                allResults << [wave: wave, label: label, status: 'FAILED', instanceId: '-']
                            }
                        }
                    }

                    // ── Summary ───────────────────────────────────────────────
                    echo '════════════════════ DEPLOYMENT SUMMARY ════════════════════'
                    allResults.each { r ->
                        echo "  ${r.status.padRight(8)} | ${r.wave} | ${r.label.padRight(32)} | instanceId=${r.instanceId}"
                    }
                    echo '═════════════════════════════════════════════════════════════'

                    def failed = allResults.findAll { it.status == 'FAILED' }
                    if (failed) {
                        error "${failed.size()} of ${allResults.size()} deployment(s) failed: ${failed.collect { it.label }.join(', ')}"
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
def deployInstance(String instanceId) {
    def body = writeJSON(returnText: true, json: [
        type          : 'HY',
        gatewayVersion: env.GATEWAY_VERSION,
        targetId      : env.FLEX_TARGET_ID,
        targetName    : env.FLEX_TARGET_NAME,
        environmentId : env.ENV_ID
    ])
    writeFile file: "deploy-${instanceId}.json", text: body
    apiCall('POST',
        "${env.ANYPOINT_BASE_URL}/proxies/xapi/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}/deployments",
        "deploy-${instanceId}.json")
}

// Apply policies from config, skipping any already applied (idempotent).
def applyPolicies(String instanceId, List policies, String appName) {
    def existingResp = apiCall('GET',
        "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}/policies",
        null)
    def existingIds = readJSON(text: existingResp).collect {
        it.template?.assetId ?: it.implementationAsset?.assetId ?: ''
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
        try {
            apiCall('POST',
                "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}/policies",
                policyFile)
            log('INFO', "Applied policy '${policy.assetId}' to ${appName}")
        } catch (err) {
            // Policy errors are non-fatal: log as WARN and continue
            log('WARN', "Policy '${policy.assetId}' failed for ${appName}: ${err.message}")
        }
    }
}

// Poll until the deployed instance is active. Fails fast on FAILED status.
def validateInstance(String instanceId, String label) {
    for (int i = 0; i < 18; i++) {
        sleep(time: 10, unit: 'SECONDS')
        def response  = apiCall('GET',
            "${env.ANYPOINT_BASE_URL}/apimanager/api/v1/organizations/${env.ORG_ID}/environments/${env.ENV_ID}/apis/${instanceId}",
            null)
        def appStatus = readJSON(text: response).deployment?.applicationStatus ?: 'UNKNOWN'
        log('INFO', "${label} check ${i + 1}/18: applicationStatus=${appStatus}")
        if (appStatus in ['STARTED', 'DEPLOYED', 'ACTIVE']) { return }
        if (appStatus == 'FAILED') { error "${label} deployment FAILED on gateway target" }
    }
    error "${label} (id=${instanceId}) did not reach an active state within 3 minutes"
}

// Authenticated Anypoint API call using PowerShell Invoke-WebRequest (Windows-native).
// Token is never logged — masked by withCredentials in the Authenticate stage.
def apiCall(String method, String url, String bodyFile) {
    def bodyArg = bodyFile ? "-Body (Get-Content -Raw '${bodyFile}')" : ''
    def raw = powershell(
        script: """
            \$ErrorActionPreference = "Stop"
            \$headers = @{
                'Authorization'    = "Bearer \$env:ANYPOINT_TOKEN"
                'X-Correlation-ID' = "\$env:CORRELATION_ID"
            }
            try {
                \$resp = Invoke-WebRequest -Method '${method}' -Uri '${url}' `
                    -Headers \$headers -ContentType 'application/json' `
                    ${bodyArg} -UseBasicParsing
                Write-Output \$resp.Content
            } catch {
                \$code = [int]\$_.Exception.Response.StatusCode
                \$msg  = \$_.ErrorDetails.Message
                if (-not \$msg) { \$msg = \$_.Exception.Message }
                throw "API call failed [${method} ${url}] HTTP \$code: \$msg"
            }
        """,
        returnStdout: true
    ).trim()
    return raw
}

// JSON-structured, correlation-aware log line.
def log(String level, String message) {
    echo "{\"level\":\"${level}\",\"correlationId\":\"${env.CORRELATION_ID}\",\"stage\":\"${env.STAGE_NAME}\",\"msg\":\"${message}\"}"
}
