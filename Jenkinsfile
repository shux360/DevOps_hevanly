pipeline {
    agent any
    
    environment {
        // Use just 'docker.io' (without username)
        DOCKER_REGISTRY = 'docker.io'
        // Your Docker Hub username
        DOCKERHUB_USERNAME = 'shux360'
        COMPOSE_PROJECT_NAME = 'devops_hevanly'
        // Credential ID (must match Jenkins credentials)
        
        // Image names (defined here for clarity)
        FRONTEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend"
        BACKEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend"
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

        stage('Build with Docker Compose') {
            steps {
                bat 'docker-compose build'
                
                // Tag images with build number
                bat """
                    docker tag ${COMPOSE_PROJECT_NAME}-frontend ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                    docker tag ${COMPOSE_PROJECT_NAME}-backend ${BACKEND_IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    passwordVariable: 'DOCKER_TOKEN',
                    usernameVariable: 'DOCKER_USERNAME',

                )]) {
                    // Method 1: Standard login
                    def loginStatus = bat(
                    script: """
                        docker logout
                        docker login -u %DOCKER_USERNAME% -p %DOCKER_TOKEN%
                    """,
                    returnStatus: true
                )
                
                // Method 2: Fallback if first fails
                if (loginStatus != 0) {
                    echo "Standard login failed, trying alternative method"
                    bat """
                        set DOCKER_CONFIG=%WORKSPACE%\\docker-config
                        mkdir %DOCKER_CONFIG%
                        echo ^{
                          \"auths\": {
                            \"https://index.docker.io/v1/\": {
                              \"auth\": \"${Base64.getEncoder().encodeToString("${DOCKER_USERNAME}:${DOCKER_TOKEN}".getBytes())}\"
                            }
                          }
                        } > %DOCKER_CONFIG%\\config.json
                        set DOCKER_CONFIG=%DOCKER_CONFIG%
                    """
                }
                }
            }
        }

        stage('Push Images') {
            steps {
                bat """
                    docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                    docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Deploy') {
            steps {
                bat """
                    docker-compose down
                    docker-compose pull
                    docker-compose up -d
                """
            }
        }
    }
}