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
        //Test email send
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
                //Both dashboard and egg-timer-app are based on HTML CSS JS.
                git url: 'https://github.com/Bn0972/UniDashboard-html-css.git', branch: 'main'
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    runCmd('echo "Running tests on Linux..."', 'echo Running tests on Windows...')
                }
            }
        }

        stage('Set Docker Context') {
            steps {
                script {
                    //if can not find desktop-linux context，switch to default
                    runCmd('docker context use default || echo "default context not found, continuing"',
                           'docker context use default || echo "default context not found, continuing"')
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}")
                }
            }
        }

        stage('Tag and Push Images') {
            steps {
                script {
                    docker.withRegistry('', 'docker-credentials') {
                        // Pull latest as previous, handle error if latest not exist
                        try {
                            // Add || true or || exit 0 to the command to avoid exceptions caused by command failure
                            runCmd(
                                "docker pull ${LATEST_IMAGE} || true",
                                "docker pull ${LATEST_IMAGE} || exit 0"
                            )
                        } catch (Exception e) {
                            echo "No latest image to tag as previous, continuing..."
                        }

                        // Tag and push previous image, skip on error
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

                        // Push new version and set as latest
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}").push()
                        runCmd(
                            "docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${VERSION} ${LATEST_IMAGE}",
                            "docker tag ${DOCKER_REGISTRY}\\${APP_NAME}:${VERSION} ${LATEST_IMAGE}"
                        )
                        runCmd(
                            "docker push ${LATEST_IMAGE}",
                            "docker push ${LATEST_IMAGE}"
                        )
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
            // when {
            //     branch 'main'
            // }
            when {
                expression {
                    // Compatible with the case where both env.BRANCH_NAME and GIT_BRANCH are empty or inconsistent
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

                        // Replace version env with :previous
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

def runCmd(String unixCmd, String windowsCmd) {
    if (isUnix()) {
        sh unixCmd
    } else {
        bat windowsCmd
    }
}
