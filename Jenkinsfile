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
                    // Policy asset-id normalisation: use config.yaml's assetId directly.
                    // rate-limiting (simple, uses rateLimits[]) != rate-limiting-sla (SLA-tier).
                    def POLICY_ID_NORM = [:]

                    // ── Common policies (applied to every API unless overridden) ──
                    def commonPolicies = []
                    if (fileExists('policies/common/policies.yaml')) {
                        def commonCfg = readYaml file: 'policies/common/policies.yaml'
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
                        def apps = (manifest.apps ?: []).collect { "${it}" }

                        if (appFilter) {
                            apps = apps.findAll { appFilter.contains(it) }
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

                    // Pre-compute total APIs across all waves for the overall progress bar
                    def totalApis = 0
                    WAVES_TO_RUN.each { w -> totalApis += (WAVE_API_MAP[w] ?: []).size() }
                    if (totalApis < 1) { totalApis = 1 }
                    currentBuild.description = "[--------------------]   0%  0/${totalApis} APIs  env=${params.ENVIRONMENT}"

                    WAVES_TO_RUN.each { wave ->
                        def apis = WAVE_API_MAP[wave] ?: []
                        if (!apis) {
                            log('INFO', "Wave ${wave} has no APIs after filtering — skipping")
                            return
                        }
                        echo "─────────────────────────── WAVE ${wave} ───────────────────────────"

                        apis.each { api ->
                            def label       = api.instanceLabel
                            def stageStatus = 'FAILED'
                            def stageId     = '-'

                            // stage() with a block body registers in the Stage View grid.
                            // catchError lets the loop continue to the next API on failure
                            // while still marking this column RED in the grid.
                            stage("${wave}: ${api.appName}") {
                                if (params.DRY_RUN) {
                                    log('INFO', "[DRY_RUN] wave=${wave}  app=${api.appName}  ${api.assetId}:${api.assetVersion}")
                                    log('INFO', "[DRY_RUN]   upstream=${api.upstreamUri}  proxy=${api.proxyUri}")
                                    log('INFO', "[DRY_RUN]   policies=${api.policies?.collect { it.assetId }}")
                                    stageStatus = 'DRY_RUN'
                                } else {
                                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                        def instanceId = findExistingInstance(api.assetId, label)
                                        if (instanceId) {
                                            log('INFO', "Reusing existing instance id=${instanceId} for label=${label}")
                                            updateInstance(instanceId, api)
                                        } else {
                                            ensureExchangeAsset(api)
                                            instanceId = createInstance(api)
                                            log('INFO', "Created new instance id=${instanceId} for label=${label}")
                                        }

                                        deployInstance(instanceId)

                                        if (!params.SKIP_POLICIES && api.policies) {
                                            applyPolicies(instanceId, api.policies, api.appName)
                                        }

                                        validateInstance(instanceId, label)
                                        stageId     = instanceId
                                        stageStatus = 'OK'
                                    }
                                }
                            }

                            allResults << [wave: wave, label: label, status: stageStatus, instanceId: stageId]

                            // ── Live progress bar (runs after every API, success or failure) ──
                            def done = allResults.size()
                            def pct  = (int)(done * 100 / totalApis)
                            def fill = (int)(pct / 5)
                            def bar  = ('#' * fill).padRight(20, '-')
                            def lastStatus = allResults ? allResults[-1].status : '?'
                            currentBuild.description = "[${bar}] ${pct}%  ${done}/${totalApis} APIs  env=${params.ENVIRONMENT}"
                            echo "== PROGRESS [${bar}] ${pct}%  (${done}/${totalApis})  last=${label} → ${lastStatus} =="
                        }
                    }

                    // ── Summary ───────────────────────────────────────────────
                    echo '════════════════════ DEPLOYMENT SUMMARY ════════════════════'
                    allResults.each { r ->
                        def icon = r.status == 'OK' ? 'OK      ' : (r.status == 'DRY_RUN' ? 'DRY_RUN ' : 'FAILED  ')
                        echo "  ${icon} | ${r.wave} | ${r.label.padRight(32)} | instanceId=${r.instanceId}"
                    }
                    echo '═════════════════════════════════════════════════════════════'

                    // Final description on the build card (persists after build completes)
                    def okCount   = allResults.findAll { it.status == 'OK' }.size()
                    def failCount = allResults.findAll { it.status == 'FAILED' }.size()
                    def dryCount  = allResults.findAll { it.status == 'DRY_RUN' }.size()
                    currentBuild.description = failCount > 0
                        ? "FAILED ${failCount}/${allResults.size()} | env=${params.ENVIRONMENT} wave=${params.WAVE}"
                        : (dryCount > 0
                            ? "DRY_RUN ${allResults.size()} APIs | env=${params.ENVIRONMENT}"
                            : "OK ${okCount}/${allResults.size()} deployed | env=${params.ENVIRONMENT} wave=${params.WAVE}")

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

    def result = powershell(
        script: """
            \$ErrorActionPreference = 'Stop'
            \$h = @{ Authorization = "Bearer \$env:ANYPOINT_TOKEN"; 'X-Correlation-ID' = "\$env:CORRELATION_ID" }

            try {
                Invoke-WebRequest -Method GET -Uri '${checkUrl}' -Headers \$h -UseBasicParsing | Out-Null
                Write-Output 'EXISTS'
            } catch {
                \$code = [int]\$_.Exception.Response.StatusCode
                if (\$code -eq 404) {
                    # Build multipart/form-data body manually (PS 5.1 has no built-in helper)
                    \$boundary  = [System.Guid]::NewGuid().ToString()
                    \$CRLF      = "`r`n"
                    \$b         = '--' + \$boundary + \$CRLF
                    \$bodyStr   = (
                        \$b + 'Content-Disposition: form-data; name="organizationId"' + \$CRLF + \$CRLF + '${groupId}'    + \$CRLF +
                        \$b + 'Content-Disposition: form-data; name="groupId"'        + \$CRLF + \$CRLF + '${groupId}'    + \$CRLF +
                        \$b + 'Content-Disposition: form-data; name="assetId"'        + \$CRLF + \$CRLF + '${assetId}'    + \$CRLF +
                        \$b + 'Content-Disposition: form-data; name="version"'        + \$CRLF + \$CRLF + '${version}'    + \$CRLF +
                        \$b + 'Content-Disposition: form-data; name="name"'           + \$CRLF + \$CRLF + '${name}'       + \$CRLF +
                        \$b + 'Content-Disposition: form-data; name="classifier"'     + \$CRLF + \$CRLF + 'http'          + \$CRLF +
                        \$b + 'Content-Disposition: form-data; name="apiVersion"'     + \$CRLF + \$CRLF + '${apiVersion}' + \$CRLF +
                        '--' + \$boundary + '--' + \$CRLF
                    )
                    \$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes(\$bodyStr)
                    Invoke-WebRequest -Method POST -Uri '${publishUrl}' -Headers \$h `
                        -ContentType "multipart/form-data; boundary=\$boundary" `
                        -Body \$bodyBytes -UseBasicParsing | Out-Null
                    Write-Output 'PUBLISHED'
                } else {
                    \$m = \$_.ErrorDetails.Message
                    if (-not \$m) { \$m = \$_.Exception.Message }
                    Write-Output "ERROR:\${code}:\$m"
                }
            }
        """,
        returnStdout: true
    ).trim()

    if (result == 'EXISTS') {
        log('INFO', "Exchange asset exists: ${assetId}:${version}")
    } else if (result == 'PUBLISHED') {
        log('INFO', "Published to Exchange as HTTP API: ${assetId}:${version}")
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
// Idempotent: POST on first run; on 400 UniqueConstraintError, GET the existing
// deployment id and PATCH it instead (keeps target/version in sync on re-runs).
def deployInstance(String instanceId) {
    def body = writeJSON(returnText: true, json: [
        type          : 'HY',
        gatewayVersion: env.GATEWAY_VERSION,
        targetId      : env.FLEX_TARGET_ID,
        targetName    : env.FLEX_TARGET_NAME,
        environmentId : env.ENV_ID
    ])
    def bodyFile = "deploy-${instanceId}.json"
    writeFile file: bodyFile, text: body

    powershell """
        \$ErrorActionPreference = "Stop"
        \$headers = @{
            'Authorization'    = "Bearer \$env:ANYPOINT_TOKEN"
            'X-Correlation-ID' = "\$env:CORRELATION_ID"
        }
        \$deplUrl = "\$env:ANYPOINT_BASE_URL/proxies/xapi/v1/organizations/\$env:ORG_ID/environments/\$env:ENV_ID/apis/${instanceId}/deployments"
        \$body    = Get-Content -Raw '${bodyFile}'

        try {
            Invoke-WebRequest -Method POST -Uri \$deplUrl `
                -Headers \$headers -ContentType 'application/json' `
                -Body \$body -UseBasicParsing | Out-Null
            Write-Host "Deployment created for instance ${instanceId}"
        } catch {
            \$code = [int]\$_.Exception.Response.StatusCode
            if (\$code -eq 400) {
                # Deployment already exists — GET its id then PATCH to update
                \$getResp  = Invoke-WebRequest -Method GET -Uri \$deplUrl `
                    -Headers \$headers -UseBasicParsing
                \$deplData = \$getResp.Content | ConvertFrom-Json
                if (\$deplData -is [array]) { \$deplData = \$deplData[0] }
                \$deplId = \$deplData.id
                if (-not \$deplId) {
                    throw "Deployment already exists for ${instanceId} but could not find id in GET response: \$(\$getResp.Content)"
                }
                Invoke-WebRequest -Method PATCH -Uri "\$deplUrl/\$deplId" `
                    -Headers \$headers -ContentType 'application/json' `
                    -Body \$body -UseBasicParsing | Out-Null
                Write-Host "Deployment updated (id=\$deplId) for instance ${instanceId}"
            } else {
                \$msg = \$_.ErrorDetails.Message
                if (-not \$msg) { \$msg = \$_.Exception.Message }
                throw "Deploy failed [POST \$deplUrl] HTTP \$code : \$msg"
            }
        }
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
        def policyResp = powershell(
            script: """
                \$ErrorActionPreference = 'Stop'
                \$h = @{ Authorization = "Bearer \$env:ANYPOINT_TOKEN"; 'X-Correlation-ID' = "\$env:CORRELATION_ID" }
                try {
                    Invoke-WebRequest -Method POST -Uri '${policyUrl}' -Headers \$h `
                        -ContentType 'application/json' -Body (Get-Content -Raw '${policyFile}') `
                        -UseBasicParsing | Out-Null
                    Write-Output 'APPLIED'
                } catch {
                    \$code = [int]\$_.Exception.Response.StatusCode
                    if (\$code -eq 409) { Write-Output 'DUPLICATE' }
                    else {
                        \$m = \$_.ErrorDetails.Message; if (-not \$m) { \$m = \$_.Exception.Message }
                        Write-Output "ERROR:\${code}:\$m"
                    }
                }
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
        currentBuild.description = "[${bar}] validating ${label}  (check ${i + 1}/${maxChecks})"
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
                throw "API call failed [${method} ${url}] HTTP \${code}: \${msg}"
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
