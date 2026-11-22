pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    parameters {
        booleanParam(
            name: 'RELEASE_APPROVED',
            defaultValue: false,
            description: 'Set to true only after the release request has been approved.'
        )
        string(
            name: 'RELEASE_REF',
            defaultValue: '',
            description: 'Approved Git branch, tag, or commit SHA to deploy.'
        )
        string(
            name: 'RELEASE_REQUEST_ID',
            defaultValue: '',
            description: 'Approved release/change request reference.'
        )
    }

    stages {
        stage('Validate Release Approval') {
            steps {
                script {
                    if (!params.RELEASE_APPROVED) {
                        error('Deployment blocked: release request has not been approved.')
                    }

                    if (!params.RELEASE_REF?.trim()) {
                        error('RELEASE_REF is required. Pass the approved Git tag, branch, or commit SHA.')
                    }

                    if (!params.RELEASE_REQUEST_ID?.trim()) {
                        error('RELEASE_REQUEST_ID is required for release traceability.')
                    }

                    if (!params.RELEASE_REF.matches('[A-Za-z0-9._/-]+') ||
                        params.RELEASE_REF.contains('..') ||
                        params.RELEASE_REF.startsWith('-')) {
                        error('RELEASE_REF contains unsupported characters.')
                    }

                    def required = [
                        'DOCKERHUB_IMAGE_REPO',
                        'DOCKERHUB_CREDENTIALS_ID',
                        'AWX_URL',
                        'AWX_JOB_TEMPLATE_ID',
                        'AWX_TOKEN_CREDENTIALS_ID',
                        'K8S_NAMESPACE',
                        'K8S_DEPLOYMENT_NAME',
                        'K8S_CONTAINER_NAME',
                        'AWX_TIMEOUT_MINUTES',
                        'AWX_POLL_SECONDS'
                    ]

                    def missing = required.findAll { key -> !env[key]?.trim() }
                    if (missing) {
                        error("Missing Jenkins Global Environment Variables: ${missing.join(', ')}")
                    }

                    if (!env.AWX_JOB_TEMPLATE_ID.matches('[0-9]+') ||
                        !env.AWX_TIMEOUT_MINUTES.matches('[0-9]+') ||
                        !env.AWX_POLL_SECONDS.matches('[0-9]+')) {
                        error('AWX_JOB_TEMPLATE_ID, AWX_TIMEOUT_MINUTES, and AWX_POLL_SECONDS must be numeric.')
                    }

                    env.AWX_BASE_URL = env.AWX_URL.replaceAll('/+$', '')
                }
            }
        }

        stage('Checkout Approved Source') {
            steps {
                checkout scm

                sh '''
                    set -eu
                    git fetch --all --tags --prune
                    if git rev-parse --verify "${RELEASE_REF}^{commit}" >/dev/null 2>&1; then
                        git checkout --detach "$RELEASE_REF"
                    elif git rev-parse --verify "origin/${RELEASE_REF}^{commit}" >/dev/null 2>&1; then
                        git checkout --detach "origin/${RELEASE_REF}"
                    else
                        echo "Approved Git ref was not found: $RELEASE_REF" >&2
                        exit 1
                    fi
                    git rev-parse --verify HEAD
                '''

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.FULL_IMAGE = "${env.DOCKERHUB_IMAGE_REPO}:${env.IMAGE_TAG}"

                    echo "Approved release: ${params.RELEASE_REF}"
                    echo "Release request: ${params.RELEASE_REQUEST_ID}"
                    echo "Image to build: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Validate Build Agent') {
            steps {
                sh '''
                    set -eu
                    command -v git
                    command -v docker
                    command -v curl
                    docker version
                    docker info
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    set -eu
                    docker build --pull -t "$FULL_IMAGE" .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${env.DOCKERHUB_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                        docker push "$FULL_IMAGE"
                    '''
                }
            }
        }

        stage('Launch AWX Deployment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: "${env.AWX_TOKEN_CREDENTIALS_ID}",
                        variable: 'AWX_TOKEN'
                    )
                ]) {
                    script {
                        def launchPayload = [
                            scm_branch: params.RELEASE_REF,
                            extra_vars: [
                                image_repo: env.DOCKERHUB_IMAGE_REPO,
                                image_tag: env.IMAGE_TAG,
                                namespace: env.K8S_NAMESPACE,
                                deployment_name: env.K8S_DEPLOYMENT_NAME,
                                container_name: env.K8S_CONTAINER_NAME,
                                release_ref: params.RELEASE_REF,
                                release_request_id: params.RELEASE_REQUEST_ID
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
                        def launchResponse = new groovy.json.JsonSlurperClassic()
                            .parseText(readFile('awx-launch-response.json'))

                        if (!launchResponse.id) {
                            error('AWX launch response did not contain a job ID.')
                        }

                        env.AWX_JOB_ID = launchResponse.id.toString()
                        echo "AWX deployment job started: ${env.AWX_JOB_ID}"
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

                            def jobStatus = new groovy.json.JsonSlurperClassic()
                                .parseText(readFile('awx-job-status.json'))
                            def status = jobStatus.status?.toString()

                            echo "AWX job status: ${status}"

                            if (status == 'successful') {
                                return true
                            }

                            if (status in ['failed', 'error', 'canceled']) {
                                error("AWX deployment failed with status: ${status}")
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
    }

    post {
        success {
            echo 'Approved release built, pushed, and deployed successfully through AWX.'
        }
        failure {
            echo 'Release pipeline failed. Review Jenkins Console Output and the AWX job.'
        }
        always {
            sh 'docker logout >/dev/null 2>&1 || true'
            cleanWs()
        }
    }
}
