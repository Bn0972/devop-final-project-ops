pipeline {
    agent any

    environment {
        // Docker Hub username
        DOCKER_REGISTRY = 'browncorry'
        APP_NAME = 'uni-dashboard'
        VERSION = "${env.BUILD_NUMBER}"
        SLACK_CHANNEL = '#deployments'
        LATEST_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}:latest"
        PREVIOUS_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}:previous"
    }

    stages {
        // Test sending email (commented out previously)
        // stage('Test Email') {
        //     steps {
        //         emailext(
        //             subject: "Jenkins Email Test",
        //             body: "If you receive this email, Jenkins mail is configured correctly.",
        //             to: "corrynn7487@gmail.com",
        //             mimeType: 'text/plain'
        //         )
        //     }
        // }

        stage('Checkout') {
            steps {
                // Clone the repository, main branch
                git url: 'https://github.com/Bn0972/UniDashboard-html-css.git', branch: 'main'
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    // Place test commands here, differentiate Linux and Windows execution
                    runCmd('echo "Running tests on Linux..."', 'echo Running tests on Windows...')
                }
            }
        }

        stage('Set Docker Context') {
            steps {
                script {
                    // Set docker context, if default not found continue without error
                    runCmd('docker context use default || echo "default context not found, continuing"',
                           'docker context use default || echo "default context not found, continuing"')
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Build two tagged images: latest and build number version
                    docker.build(LATEST_IMAGE)
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}")
                }
            }
        }

        stage('Tag and Push Images') {
            steps {
                script {
                    docker.withRegistry('', 'docker-credentials') {
                        // Push the version-tagged image first
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}").push()

                        // Push the latest tag image next
                        docker.image(LATEST_IMAGE).push()

                        // Try to pull latest tag image; if not exist, ignore errors
                        try {
                            runCmd(
                                "docker pull ${LATEST_IMAGE} || true",
                                "docker pull ${LATEST_IMAGE} || exit 0"
                            )
                        } catch (Exception e) {
                            echo "No latest image to tag as previous, skipping tag of previous image."
                        }

                        // If pull succeeded, tag it as previous and push
                        try {
                            runCmd(
                                "docker tag ${LATEST_IMAGE} ${PREVIOUS_IMAGE}",
                                "docker tag ${LATEST_IMAGE} ${PREVIOUS_IMAGE}"
                            )
                            runCmd(
                                "docker push ${PREVIOUS_IMAGE}",
                                "docker push ${PREVIOUS_IMAGE}"
                            )
                        } catch (Exception e) {
                            echo "Tagging or pushing previous image skipped."
                        }
                    }
                }
            }
        }

        stage('Debug Workspace') {
            steps {
                script {
                    echo "Current env.BRANCH_NAME: ${env.BRANCH_NAME}"
                    echo "Current env.GIT_BRANCH: ${env.GIT_BRANCH}"
                    if (isUnix()) {
                        sh 'pwd && ls -l'
                    } else {
                        bat 'cd && dir'
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    try {
                        runCmd('docker-compose -f docker-compose.staging.yml up -d', 'docker-compose -f docker-compose.staging.yml up -d')
                        sleep 30
                        runCmd('curl -f http://localhost:8081 || exit 1', 'curl -f http://localhost:8081 || exit /b 1')
                    } catch (Exception e) {
                        runCmd('docker-compose -f docker-compose.staging.yml down', 'docker-compose -f docker-compose.staging.yml down')
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }

        stage('Deploy to Production') {
            when {
                expression {
                    return env.BRANCH_NAME == 'main' || env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main'
                }
            }
            steps {
                script {
                    try {
                        runCmd('docker-compose -f docker-compose.prod.yml up -d', 'docker-compose -f docker-compose.prod.yml up -d')
                        sleep 30
                        runCmd('curl -f http://localhost:3100 || exit 1', 'curl -f http://localhost:3100 || exit /b 1')
                    } catch (Exception e) {
                        echo "Production deployment failed. Rolling back to previous image..."

                        // Roll back to previous tag
                        env.VERSION = "previous"

                        runCmd('docker-compose -f docker-compose.prod.yml down', 'docker-compose -f docker-compose.prod.yml down')
                        runCmd('docker-compose -f docker-compose.prod.yml up -d', 'docker-compose -f docker-compose.prod.yml up -d')

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                slackSend(
                    channel: "${SLACK_CHANNEL}",
                    color: 'good',
                    message: "Deployment Successful: ${APP_NAME} v${VERSION}"
                )
                emailext(
                    subject: "Deployment Successful: ${APP_NAME} v${VERSION}",
                    body: 'The deployment was successful.',
                    to: "corrynn7487@gmail.com",
                    from: "corrynn7487@gmail.com", 
                    replyTo: "corrynn7487@gmail.com",
                    mimeType: 'text/plain'
                )
            }
        }
        failure {
            script {
                slackSend(
                    channel: "${SLACK_CHANNEL}",
                    color: 'danger',
                    message: "Deployment Failed: ${APP_NAME} v${VERSION}"
                )
                emailext(
                    subject: "Deployment Failed: ${APP_NAME} v${VERSION}",
                    body: 'The deployment failed. Please check Jenkins for details.',
                    to: "corrynn7487@gmail.com",
                    from: "corrynn7487@gmail.com", 
                    replyTo: "corrynn7487@gmail.com",
                    mimeType: 'text/plain'
                )
            }
        }
    }
}

// Helper function to run shell on Unix or bat on Windows
def runCmd(String unixCmd, String windowsCmd) {
    if (isUnix()) {
        sh unixCmd
    } else {
        bat windowsCmd
    }
}
