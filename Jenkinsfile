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
            description: 'Set true only after release approval.'
        )

        string(
            name: 'RELEASE_REF',
            defaultValue: '',
            description: 'Approved Git branch, tag, or commit SHA.'
        )

        string(
            name: 'RELEASE_REQUEST_ID',
            defaultValue: '',
            description: 'Approved release/change request ID.'
        )
    }

    stages {
        stage('Validate Release Approval and Configuration') {
            steps {
                script {
                    if (!params.RELEASE_APPROVED) {
                        error('Deployment blocked: release not approved.')
                    }

                    if (!params.RELEASE_REF?.trim()) {
                        error('RELEASE_REF is required.')
                    }

                    if (!params.RELEASE_REQUEST_ID?.trim()) {
                        error('RELEASE_REQUEST_ID is required.')
                    }

                    if (!params.RELEASE_REF.matches('[A-Za-z0-9._/-]+') ||
                        params.RELEASE_REF.contains('..') ||
                        params.RELEASE_REF.startsWith('-')) {
                        error('Invalid RELEASE_REF.')
                    }

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

                    def missingVariables = requiredVariables.findAll { key ->
                        !env[key]?.trim()
                    }

                    if (missingVariables) {
                        error("Missing Jenkins Global Environment Variables: ${missingVariables.join(', ')}")
                    }

                    if (!(env.AWX_JOB_TEMPLATE_ID ==~ /[0-9]+/)) {
                        error('AWX_JOB_TEMPLATE_ID must be numeric.')
                    }

                    if (!(env.AWX_TIMEOUT_MINUTES ==~ /[0-9]+/) ||
                        env.AWX_TIMEOUT_MINUTES.toInteger() <= 0) {
                        error('AWX_TIMEOUT_MINUTES must be a positive integer.')
                    }

                    if (!(env.AWX_POLL_SECONDS ==~ /[0-9]+/) ||
                        env.AWX_POLL_SECONDS.toInteger() <= 0) {
                        error('AWX_POLL_SECONDS must be a positive integer.')
                    }

                    if (!(env.DOCKERHUB_IMAGE_REPO ==~ /[A-Za-z0-9._/-]+/)) {
                        error('DOCKERHUB_IMAGE_REPO contains unsupported characters.')
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
                        echo "Approved Git ref not found: $RELEASE_REF"
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

                    echo "Release request: ${params.RELEASE_REQUEST_ID}"
                    echo "Approved Git ref: ${params.RELEASE_REF}"
                    echo "Image to publish: ${env.FULL_IMAGE}"
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

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build --pull -t "$FULL_IMAGE" .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
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
                        credentialsId: 'awx-api-token',
                        variable: 'AWX_TOKEN'
                    )
                ]) {
                    script {
                        def launchPayload = [
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
                            error('AWX response did not contain a job ID.')
                        }

                        env.AWX_JOB_ID = launchResponse.id.toString()
                        echo "AWX Job started: ${env.AWX_JOB_ID}"
                    }

                    // Declarative Pipeline requires Groovy control logic inside script {}.
                    script {
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
                                echo "AWX Job status: ${status}"

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
    }

    post {
        success {
            echo 'Release built, pushed and deployed successfully through AWX.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins Console Output and the AWX job.'
        }

        always {
            sh 'docker logout >/dev/null 2>&1 || true'
            cleanWs()
        }
    }
}
