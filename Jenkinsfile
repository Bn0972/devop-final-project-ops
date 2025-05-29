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
                sh 'echo "Running tests..."'
                // Add your test commands here
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
                    docker.withRegistry('https://${DOCKER_REGISTRY}', 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}").push()
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                script {
                    try {
                        sh "docker-compose -f docker-compose.staging.yml up -d"
                        // Wait for health check
                        sleep 30
                        // Verify deployment
                        sh "curl -f http://staging.egg-timer-app/health || exit 1"
                    } catch (Exception e) {
                        // Rollback on failure
                        sh "docker-compose -f docker-compose.staging.yml down"
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
                        sh "docker-compose -f docker-compose.prod.yml up -d"
                        // Wait for health check
                        sleep 30
                        // Verify deployment
                        sh "curl -f http://prod.egg-timer-app/health || exit 1"
                    } catch (Exception e) {
                        // Rollback on failure
                        sh "docker-compose -f docker-compose.prod.yml down"
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
                // Send success notifications
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
                // Send failure notifications
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