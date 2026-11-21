
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        export FULL_IMAGE="$DOCKER_USER/$DOCKERHUB_REPO:$IMAGE_TAG"

                        docker build -t "$FULL_IMAGE" .
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
                        FULL_IMAGE="$DOCKER_USER/$DOCKERHUB_REPO:$IMAGE_TAG"

                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" --password-stdin

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
                        writeFile(
                            file: 'awx-extra-vars.json',
                            text: groovy.json.JsonOutput.toJson([
                                image_repo: "${DOCKER_USER}/${DOCKERHUB_REPO}",
                                image_tag: env.IMAGE_TAG
                            ])
                        )
                    }

                    sh '''
                        curl --fail --show-error \
                            -X POST \
                            "$AWX_URL/api/v2/job_templates/$AWX_JOB_TEMPLATE_ID/launch/" \
                            -H "Authorization: Bearer $AWX_TOKEN" \
                            -H "Content-Type: application/json" \
                            --data-binary @awx-extra-vars.json
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Docker image pushed and AWX deployment triggered successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins Console Output.'
        }
    }
}
