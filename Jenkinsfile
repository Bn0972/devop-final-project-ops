pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'browncorry'
        APP_NAME = 'egg-timer-app'
        VERSION = "${env.BUILD_NUMBER}"
        SLACK_CHANNEL = '#deployments'
        EMAIL_RECIPIENTS = 'corrynn7487@gmail.com'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    runCmd('echo "Running tests on Linux..."', 'echo Running tests on Windows...')
                    // 示例：可以替换为测试命令
                    // runCmd('./run_tests.sh', 'run_tests.bat')
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
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}").push()
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
                            'curl -f http://staging.egg-timer-app/health || exit 1',
                            'curl -f http://staging.egg-timer-app/health || exit /b 1'
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
                            'curl -f http://prod.egg-timer-app/health || exit 1',
                            'curl -f http://prod.egg-timer-app/health || exit /b 1'
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
                emailext (
                    subject: "Deployment Successful: ${APP_NAME} v${VERSION}",
                    body: "The deployment was successful.",
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
                emailext (
                    subject: "Deployment Failed: ${APP_NAME} v${VERSION}",
                    body: "The deployment failed. Please check Jenkins for details.",
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
