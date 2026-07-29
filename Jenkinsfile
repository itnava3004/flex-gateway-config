// ============================================================
// Jenkinsfile
// Project : EAI-3541669 — Flex Gateway Config
// Purpose : CI/CD pipeline for validating and deploying Flex
//           Gateway API configurations across all environments
// ============================================================

pipeline {

    agent {
        docker {
            image 'mikefarah/yq:4'   // yq is the only tooling dependency
            args  '--entrypoint='
        }
    }

    // --- Pipeline-level options ---
    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    // --- Parameters (override via Jenkins UI or upstream trigger) ---
    parameters {
        choice(
            name: 'TARGET_ENV',
            choices: ['dev', 'test', 'qa', 'preprod', 'prod'],
            description: 'Target deployment environment'
        )
        choice(
            name: 'WAVE',
            choices: ['all', 'R1', 'R2'],
            description: 'Release wave to deploy (all = deploy every active API)'
        )
        booleanParam(
            name: 'DRY_RUN',
            defaultValue: true,
            description: 'When true, validate and merge configs but do NOT call flexctl apply'
        )
    }

    // --- Environment variables (credentials from Jenkins Credentials Store) ---
    environment {
        ANYPOINT_CLIENT_ID     = credentials('anypoint-client-id')
        ANYPOINT_CLIENT_SECRET = credentials('anypoint-client-secret')
        FLEX_ORG_ID            = credentials('flex-org-id')
        FLEX_ENV_ID            = credentials("flex-env-id-${params.TARGET_ENV}")
    }

    stages {

        // ----------------------------------------------------------
        stage('Checkout') {
        // ----------------------------------------------------------
            steps {
                checkout scm
                echo "Branch  : ${env.BRANCH_NAME}"
                echo "Commit  : ${env.GIT_COMMIT}"
                echo "Env     : ${params.TARGET_ENV}"
                echo "Wave    : ${params.WAVE}"
                echo "Dry-run : ${params.DRY_RUN}"
            }
        }

        // ----------------------------------------------------------
        stage('Validate YAML') {
        // ----------------------------------------------------------
            steps {
                sh '''
                    chmod +x src/deploy/validate.sh
                    src/deploy/validate.sh
                '''
            }
            post {
                failure {
                    echo '[ERROR] YAML validation failed — halting pipeline.'
                }
            }
        }

        // ----------------------------------------------------------
        stage('Deploy — Non-Prod') {
        // ----------------------------------------------------------
            when {
                allOf {
                    expression { params.TARGET_ENV in ['dev', 'test', 'qa'] }
                    expression { params.DRY_RUN == false }
                }
            }
            steps {
                sh """
                    chmod +x src/deploy/deploy.sh
                    src/deploy/deploy.sh ${params.TARGET_ENV} ${params.WAVE}
                """
            }
        }

        // ----------------------------------------------------------
        stage('Deploy — Pre-Prod') {
        // ----------------------------------------------------------
            when {
                allOf {
                    expression { params.TARGET_ENV == 'preprod' }
                    expression { params.DRY_RUN == false }
                    branch 'main'
                }
            }
            steps {
                sh """
                    chmod +x src/deploy/deploy.sh
                    src/deploy/deploy.sh preprod ${params.WAVE}
                """
            }
        }

        // ----------------------------------------------------------
        stage('Approval Gate — Production') {
        // ----------------------------------------------------------
            when {
                allOf {
                    expression { params.TARGET_ENV == 'prod' }
                    expression { params.DRY_RUN == false }
                    branch 'main'
                }
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Approve production deployment? (Wave: ${params.WAVE})",
                          ok: 'Deploy to Production',
                          submitter: 'integration-leads'
                }
            }
        }

        // ----------------------------------------------------------
        stage('Deploy — Production') {
        // ----------------------------------------------------------
            when {
                allOf {
                    expression { params.TARGET_ENV == 'prod' }
                    expression { params.DRY_RUN == false }
                    branch 'main'
                }
            }
            steps {
                sh """
                    chmod +x src/deploy/deploy.sh
                    src/deploy/deploy.sh prod ${params.WAVE}
                """
            }
        }

        // ----------------------------------------------------------
        stage('Dry Run Summary') {
        // ----------------------------------------------------------
            when {
                expression { params.DRY_RUN == true }
            }
            steps {
                echo "[DRY RUN] No changes were applied. Re-run with DRY_RUN=false to deploy."
                sh """
                    chmod +x src/deploy/deploy.sh
                    src/deploy/deploy.sh ${params.TARGET_ENV} ${params.WAVE}
                """
            }
        }

    } // end stages

    post {
        always {
            echo "Pipeline finished — build #${env.BUILD_NUMBER}"
        }
        success {
            echo "[SUCCESS] EAI-3541669 deployment pipeline completed."
        }
        failure {
            echo "[FAILURE] Pipeline failed. Check logs above."
            // Add mail/Slack notification step here if required
        }
    }

} // end pipeline
