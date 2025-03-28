pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/shux360'
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/havenly_frontend"
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/havenly_backend"
        DOCKER_CREDENTIALS = 'dockerhub-cred'
        
        // Add these credentials (set up in Jenkins credentials manager)
        MONGODB_URI = credentials('mongodb-uri')
        JWT_SECRET = credentials('jwt-secret')
    }

    stages {
        stage('SCM Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Frontend Dependencies') {
                    steps {
                        dir('havenly_frontend-main') {
                            sh 'npm install'
                        }
                    }
                }
                stage('Backend Dependencies') {
                    steps {
                        dir('havenly_backend-main') {
                            sh 'npm install'
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend Image') {
                    steps {
                        dir('havenly_frontend-main') {
                            sh "docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} ."
                        }
                    }
                }
                stage('Build Backend Image') {
                    steps {
                        dir('havenly_backend-main') {
                            sh """
                                docker build \
                                --build-arg MONGODB_URI=\${MONGODB_URI} \
                                --build-arg JWT_SECRET=\${JWT_SECRET} \
                                -t ${BACKEND_IMAGE}:${BUILD_NUMBER} .
                            """
                        }
                    }
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}", 
                    passwordVariable: 'DOCKER_PASSWORD', 
                    usernameVariable: 'DOCKER_USERNAME'
                )]) {
                    sh "echo \$DOCKER_PASSWORD | docker login ${DOCKER_REGISTRY} -u \$DOCKER_USERNAME --password-stdin"
                }
            }
        }

        stage('Push Docker Images') {
            parallel {
                stage('Push Frontend Image') {
                    steps {
                        sh "docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                    }
                }
                stage('Push Backend Image') {
                    steps {
                        sh "docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Deploy to Development') {
            steps {
                sh """
                    sed -i 's|${FRONTEND_IMAGE}:[^ ]*|${FRONTEND_IMAGE}:${BUILD_NUMBER}|g' docker-compose.yml
                    sed -i 's|${BACKEND_IMAGE}:[^ ]*|${BACKEND_IMAGE}:${BUILD_NUMBER}|g' docker-compose.yml
                    docker-compose up -d --build
                """
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${FRONTEND_IMAGE}:${BUILD_NUMBER} || true
                docker rmi ${BACKEND_IMAGE}:${BUILD_NUMBER} || true
            """
            cleanWs()
        }
    }
}