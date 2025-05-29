pipeline {
    agent any

    environment {
        // Docker Hub username
        DOCKER_REGISTRY = 'browncorry'
        APP_NAME = 'uni-dashboard'
        VERSION = "${env.BUILD_NUMBER}"
        SLACK_CHANNEL = '#deployments'
        EMAIL_RECIPIENTS = 'corrynn7487@gmail.com'
    }

    stages {
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

        stage('Push to Registry') {
            steps {
                script {
                        // empty string means Docker Hub default
                    docker.withRegistry('', 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}").push()
                    }
                }
            }
        }

        stage('Debug Workspace') {
            steps {
                script {
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
                        runCmd(
                            'docker-compose -f docker-compose.staging.yml up -d',
                            'docker-compose -f docker-compose.staging.yml up -d'
                        )
                        sleep 30
                        runCmd(
                            'curl -f http://localhost:8081/health || exit 1',
                            'curl -f http://localhost:8081/health || exit /b 1'
                        )
                    } catch (Exception e) {
                        runCmd(
                            'docker-compose -f docker-compose.staging.yml down',
                            'docker-compose -f docker-compose.staging.yml down'
                        )
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    try {
                        runCmd(
                            'docker-compose -f docker-compose.prod.yml up -d',
                            'docker-compose -f docker-compose.prod.yml up -d'
                        )
                        sleep 30
                        runCmd(
                            'curl -f http://localhost:80/health || exit 1',
                            'curl -f http://localhost:80/health || exit /b 1'
                        )
                    } catch (Exception e) {
                        runCmd(
                            'docker-compose -f docker-compose.prod.yml down',
                            'docker-compose -f docker-compose.prod.yml down'
                        )
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
                    to: "${EMAIL_RECIPIENTS}"
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
                    to: "${EMAIL_RECIPIENTS}"
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