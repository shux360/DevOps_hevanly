pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/shux360'
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/havenly_frontend"
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/havenly_backend"
        DOCKER_CREDENTIALS = 'dockerhub-cred'
        GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('havenly_backend-main') {
                            sh 'npm install'
                            sh 'npm test'
                        }
                    }
                }
                stage('Frontend Tests') {
                    steps {
                        dir('havenly_frontend-main') {
                            sh 'npm install'
                            sh 'npm test'
                        }
                    }
                }
            }
        }

        stage('Build') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        script {
                            withCredentials([string(credentialsId: 'vite-api-url', variable: 'VITE_API_URL')]) {
                                dir('havenly_frontend-main') {
                                    sh """
                                        docker build \
                                            --build-arg VITE_API_URL=${VITE_API_URL} \
                                            -t ${FRONTEND_IMAGE}:${GIT_COMMIT_SHORT} \
                                            -t ${FRONTEND_IMAGE}:latest \
                                            .
                                    """
                                }
                            }
                        }
                    }
                }
                stage('Build Backend') {
                    steps {
                        script {
                            withCredentials([
                                string(credentialsId: 'jwt-secret', variable: 'JWT_SECRET'),
                                string(credentialsId: 'mongodb-uri', variable: 'MONGODB_URI')
                            ]) {
                                dir('havenly_backend-main') {
                                    sh """
                                        docker build \
                                            --build-arg MONGODB_URI=${MONGODB_URI} \
                                            --build-arg JWT_SECRET=${JWT_SECRET} \
                                            -t ${BACKEND_IMAGE}:${GIT_COMMIT_SHORT} \
                                            -t ${BACKEND_IMAGE}:latest \
                                            .
                                    """
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_CREDENTIALS,
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                        sh "docker push ${FRONTEND_IMAGE}:${GIT_COMMIT_SHORT}"
                        sh "docker push ${FRONTEND_IMAGE}:latest"
                        sh "docker push ${BACKEND_IMAGE}:${GIT_COMMIT_SHORT}"
                        sh "docker push ${BACKEND_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh "docker-compose down || true"
                    sh "docker-compose up -d"
                }
            }
        }
    }

    post {
        always {
            script {
                sh "docker logout || true"
                cleanWs()
            }
        }
        success {
            slackSend(color: 'good', message: "Build ${BUILD_NUMBER} succeeded!")
        }
        failure {
            slackSend(color: 'danger', message: "Build ${BUILD_NUMBER} failed!")
        }
    }
}