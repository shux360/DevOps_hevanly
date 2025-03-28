pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/shux360'  // Replace with your registry
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/devops_hevanly-frontend"
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/devops_hevanly-backend"
        DOCKER_CREDENTIALS = 'dockerhub-cred'  // Replace with your credentials ID
        // MONGODB_URI = credentials('mongodb-uri')
        // JWT_SECRET = credentials('jwt-secret')

    }

    stages {
        stage('Github Trigger') {
            steps {
                script {
                    properties([pipelineTriggers([githubPush()])])
                }
            }
        }

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
                            bat 'npm install'
                        }
                    }
                }
                stage('Backend Dependencies') {
                    steps {
                        dir('havenly_backend-main') {
                            bat 'npm install'
                        }
                    }
                }
            }
        }

        // stage('Run Tests') {
        //     parallel {
        //         stage('Frontend Tests') {
        //             steps {
        //                 dir('havenly_frontend-main') {
        //                     bat 'npm run test'
        //                 }
        //             }
        //         }
        //         stage('Backend Tests') {
        //             steps {
        //                 dir('havenly_backend-main') {
        //                     bat 'npm run test'
        //                 }
        //             }
        //         }
        //     }
        // }

        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend Image') {
                    steps {
                        dir('havenly_frontend-main') {
                            script {
                                sh "docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                            }
                        }
                    }
                }
                stage('Build Backend Image') {
                    steps {
                        dir('havenly_backend-main') {
                            script {
                                sh "docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER}"
                            }
                        }
                    }
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId:'dockerhub-cred', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    bat "echo $DOCKER_PASSWORD | docker login $DOCKER_REGISTRY -u $DOCKER_USERNAME --password-stdin"
                }
            }
        }

        stage('Push Docker Images') {
            parallel {
                stage('Push Frontend Image') {
                    steps {
                        bat "docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                    }
                }
                stage('Push Backend Image') {
                    steps {
                        bat "docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Deploy to Development') {
            steps {
                script {
                    // Update docker-compose with new image tags
                    bat """
                        sed -i 's|${FRONTEND_IMAGE}:[^ ]*|${FRONTEND_IMAGE}:${BUILD_NUMBER}|g' docker-compose.yml
                        sed -i 's|${BACKEND_IMAGE}:[^ ]*|${BACKEND_IMAGE}:${BUILD_NUMBER}|g' docker-compose.yml
                        docker-compose up -d
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean up Docker images
            bat """
                docker rmi ${FRONTEND_IMAGE}:${BUILD_NUMBER} || true
                docker rmi ${BACKEND_IMAGE}:${BUILD_NUMBER} || true
            """
            // Clean workspace
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
            // Add notification steps here (email, Slack, etc.)
        }
    }
}
