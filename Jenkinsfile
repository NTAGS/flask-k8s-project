
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    stages {

        stage('Validate Configuration') {
            steps {
                script {
                    def requiredVariables = [
                        'DOCKERHUB_IMAGE_REPO',
                        'AWX_URL',
                        'AWX_JOB_TEMPLATE_ID',
                        'K8S_NAMESPACE',
                        'K8S_DEPLOYMENT_NAME',
                        'K8S_CONTAINER_NAME',
                        'AWX_TIMEOUT_MINUTES',
                        'AWX_POLL_SECONDS'
                    ]

                    def missingVariables = requiredVariables.findAll {
                        !env."${it}"?.trim()
                    }

                    if (missingVariables) {
                        error("Missing Jenkins Global Environment Variables: ${missingVariables.join(', ')}")
                    }

                    if (!(env.AWX_JOB_TEMPLATE_ID ==~ '[0-9]+')) {
                        error('AWX_JOB_TEMPLATE_ID must be numeric.')
                    }

                    if (!(env.AWX_TIMEOUT_MINUTES ==~ '[0-9]+') ||
                        env.AWX_TIMEOUT_MINUTES.toInteger() <= 0) {
                        error('AWX_TIMEOUT_MINUTES must be a positive integer.')
                    }

                    if (!(env.AWX_POLL_SECONDS ==~ '[0-9]+') ||
                        env.AWX_POLL_SECONDS.toInteger() <= 0) {
                        error('AWX_POLL_SECONDS must be a positive integer.')
                    }

                    if (!(env.DOCKERHUB_IMAGE_REPO ==~ '[A-Za-z0-9._/-]+')) {
                        error('Invalid Docker Hub image repository.')
                    }

                    if (!(env.AWX_URL.startsWith('http://') ||
                          env.AWX_URL.startsWith('https://'))) {
                        error('AWX_URL must start with http:// or https://')
                    }

                    env.AWX_BASE_URL = env.AWX_URL.replaceAll('/+$', '')
                }
            }
        }

        stage('Checkout Release Branch') {
            steps {
                checkout scm

                sh '''
                    set -eu

                    echo "Checking out release branch source..."

                    git status --short
                    git log -1 --oneline
                '''

                script {
                    env.SOURCE_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.FULL_IMAGE = "${env.DOCKERHUB_IMAGE_REPO}:${env.IMAGE_TAG}"
                    env.RELEASE_REQUEST_ID = "JENKINS-${env.BUILD_NUMBER}"

                    echo "Release branch: release"
                    echo "Source commit: ${env.SOURCE_COMMIT}"
                    echo "Release ID: ${env.RELEASE_REQUEST_ID}"
                    echo "Image: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Validate Jenkins Agent') {
            steps {
                sh '''
                    set -eu

                    command -v git
                    command -v curl
                    command -v python3

                    echo "Jenkins agent validation successful."
                    echo "Docker is not required on Jenkins."
                '''
            }
        }

        stage('Launch AWX Deployment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'awx-api-token',
                        variable: 'AWX_TOKEN'
                    )
                ]) {

                    script {
                        def launchPayload = [
                            extra_vars: [
                                image_repo: env.DOCKERHUB_IMAGE_REPO,
                                image_tag: env.IMAGE_TAG,
                                source_commit: env.SOURCE_COMMIT,
                                namespace: env.K8S_NAMESPACE,
                                deployment_name: env.K8S_DEPLOYMENT_NAME,
                                container_name: env.K8S_CONTAINER_NAME,
                                release_ref: 'release',
                                release_request_id: env.RELEASE_REQUEST_ID
                            ]
                        ]

                        writeFile(
                            file: 'awx-launch-payload.json',
                            text: groovy.json.JsonOutput.toJson(launchPayload)
                        )
                    }

                    sh '''
                        set -eu
                        set +x

                        curl --fail --silent --show-error \
                            --request POST \
                            "$AWX_BASE_URL/api/v2/job_templates/$AWX_JOB_TEMPLATE_ID/launch/" \
                            --header "Authorization: Bearer $AWX_TOKEN" \
                            --header "Content-Type: application/json" \
                            --data-binary @awx-launch-payload.json \
                            --output awx-launch-response.json
                    '''

                    script {
                        env.AWX_JOB_ID = sh(
                            script: '''
                                python3 -c "import json; print(json.load(open('awx-launch-response.json'))['id'])"
                            ''',
                            returnStdout: true
                        ).trim()

                        if (!(env.AWX_JOB_ID ==~ '[0-9]+')) {
                            error('AWX response did not contain a valid job ID.')
                        }

                        echo "AWX Job started: ${env.AWX_JOB_ID}"
                    }

                    timeout(
                        time: env.AWX_TIMEOUT_MINUTES.toInteger(),
                        unit: 'MINUTES'
                    ) {
                        waitUntil {
                            sh '''
                                set -eu
                                set +x

                                curl --fail --silent --show-error \
                                    "$AWX_BASE_URL/api/v2/jobs/$AWX_JOB_ID/" \
                                    --header "Authorization: Bearer $AWX_TOKEN" \
                                    --header "Content-Type: application/json" \
                                    --output awx-job-status.json
                            '''

                            env.AWX_JOB_STATUS = sh(
                                script: '''
                                    python3 -c "import json; print(json.load(open('awx-job-status.json'))['status'])"
                                ''',
                                returnStdout: true
                            ).trim()

                            echo "AWX Job status: ${env.AWX_JOB_STATUS}"

                            if (env.AWX_JOB_STATUS == 'successful') {
                                return true
                            }

                            if (env.AWX_JOB_STATUS in [
                                'failed',
                                'error',
                                'canceled'
                            ]) {
                                error("AWX deployment failed: ${env.AWX_JOB_STATUS}")
                            }

                            sleep(
                                time: env.AWX_POLL_SECONDS.toInteger(),
                                unit: 'SECONDS'
                            )

                            return false
                        }
                    }
                }
            }
        }

        stage('Release Deployment Summary') {
            steps {
                echo """
                RELEASE DEPLOYMENT SUCCESSFUL

                Branch: release
                Commit: ${env.SOURCE_COMMIT}
                Release ID: ${env.RELEASE_REQUEST_ID}
                Docker Image: ${env.FULL_IMAGE}
                AWX Job: ${env.AWX_JOB_ID}
                Kubernetes Namespace: ${env.K8S_NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo 'Release successfully built, pushed and deployed through AWX.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins Console Output and the AWX job.'
        }

        always {
            cleanWs()
        }
    }
}
