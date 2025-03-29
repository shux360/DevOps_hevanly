pipeline {
    agent any
    
    environment {
        // Use just 'docker.io' as the registry (without your username)
        DOCKER_REGISTRY = 'docker.io'  
        DOCKERHUB_USERNAME = 'shux360'
        COMPOSE_PROJECT_NAME = 'devops_hevanly'
        // Make sure this credential uses your access token as password
        DOCKER_CREDENTIALS = 'dockerhub-cred'  
    }

    stages {
        stage('Verify Docker Setup') {
            steps {
                script {
                    try {
                        // Check Docker is working
                        bat 'docker --version'
                        // Verify network connectivity to Docker Hub
                        bat 'ping registry-1.docker.io -n 2'
                    } catch (Exception e) {
                        error("Docker setup verification failed: ${e.message}")
                    }
                }
            }
        }

        stage('Build Images') {
            steps {
                bat 'docker-compose build'
            }
        }

        stage('Tag Images') {
            steps {
                bat """
                    docker tag ${COMPOSE_PROJECT_NAME}-frontend ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend:${BUILD_NUMBER}
                    docker tag ${COMPOSE_PROJECT_NAME}-backend ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend:${BUILD_NUMBER}
                """
            }
        }

        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_CREDENTIALS,
                        passwordVariable: 'DOCKER_TOKEN',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        // First attempt - standard login
                        def loginAttempt = bat(
                            script: "docker login -u %DOCKER_USERNAME% -p %DOCKER_TOKEN%",
                            returnStatus: true
                        )
                        
                        // Second attempt - alternative method if first fails
                        if (loginAttempt != 0) {
                            echo "Standard login failed, trying alternative method"
                            bat """
                                set DOCKER_CONFIG=%TEMP%\\docker-config
                                mkdir %DOCKER_CONFIG%
                                echo ^{
                                  \"auths\": {
                                    \"https://index.docker.io/v1/\": {
                                      \"auth\": \"${Base64.getEncoder().encodeToString("${DOCKER_USERNAME}:${DOCKER_TOKEN}".getBytes())}\"
                                    }
                                  }
                                } > %DOCKER_CONFIG%\\config.json
                                set DOCKER_CONFIG=%DOCKER_CONFIG%
                                docker login -u %DOCKER_USERNAME% --password-stdin <<< %DOCKER_TOKEN%
                            """
                        }
                        
                        // Verify login succeeded
                        bat 'docker info | find "Username"'
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                bat """
                    docker push ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend:${BUILD_NUMBER}
                    docker push ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend:${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        always {
            script {
                // Cleanup commands with error suppression
                bat(script: 'docker logout || true', returnStatus: true)
                bat(script: 'docker rmi ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend:${BUILD_NUMBER} || true', returnStatus: true)
                bat(script: 'docker rmi ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend:${BUILD_NUMBER} || true', returnStatus: true)
                cleanWs()
            }
        }
    }
}