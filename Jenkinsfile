
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_REPO = 'YOUR_DOCKERHUB_REPOSITORY'
        AWX_URL = 'http://192.168.56.6:30474'
        AWX_JOB_TEMPLATE_ID = 'YOUR_AWX_JOB_TEMPLATE_ID'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''

                    if (!branch) {
                        branch = sh(
                            script: 'git rev-parse --abbrev-ref HEAD',
                            returnStdout: true
                        ).trim()
                    }

                    env.SOURCE_BRANCH = branch.tokenize('/').last()

                    if (env.SOURCE_BRANCH != 'release') {
                        error("This pipeline supports only release branch. Found: ${env.SOURCE_BRANCH}")
                    }

                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Branch: ${env.SOURCE_BRANCH}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Validate Docker Builder') {
            steps {
                sh '''
                    command -v docker
                    docker version
                    docker info
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e

                        FULL_IMAGE="$DOCKER_USER/$DOCKERHUB_REPO:$IMAGE_TAG"

                        docker build -t "$FULL_IMAGE" .

                        echo "Image built: $FULL_IMAGE"
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e
                        set +x

                        FULL_IMAGE="$DOCKER_USER/$DOCKERHUB_REPO:$IMAGE_TAG"

                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        docker push "$FULL_IMAGE"

                        docker logout
                    '''
                }
            }
        }

        stage('Trigger AWX Deployment') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    ),
                    string(
                        credentialsId: 'awx-api-token',
                        variable: 'AWX_TOKEN'
                    )
                ]) {
                    script {
                        def payload = groovy.json.JsonOutput.toJson([
                            extra_vars: [
                                image_repo: "${env.DOCKER_USER}/${env.DOCKERHUB_REPO}",
                                image_tag : env.IMAGE_TAG
                            ]
                        ])

                        writeFile(
                            file: 'awx-payload.json',
                            text: payload
                        )
                    }

                    sh '''
                        set -e
                        set +x

                        curl --fail --show-error \
                            -X POST \
                            "$AWX_URL/api/v2/job_templates/$AWX_JOB_TEMPLATE_ID/launch/" \
                            -H "Authorization: Bearer $AWX_TOKEN" \
                            -H "Content-Type: application/json" \
                            --data-binary @awx-payload.json

                        rm -f awx-payload.json
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Docker image pushed and AWX deployment job submitted successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins Console Output.'
        }

        always {
            cleanWs()
        }
    }
}
