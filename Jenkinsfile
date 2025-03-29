pipeline {
    agent any
    
    environment {
        // Docker configuration
        DOCKER_REGISTRY = 'index.docker.io'  // More reliable endpoint
        DOCKERHUB_USERNAME = 'shux360'       // Your Docker Hub username
        COMPOSE_PROJECT_NAME = 'devops_hevanly'
        
        // Jenkins credential ID for Docker Hub (username + access token)
        DOCKER_CREDENTIALS = 'dockerhub-token'  
        
        // Uncomment and configure if behind corporate proxy/firewall:
        // http_proxy = "http://yourproxy:port"
        // https_proxy = "http://yourproxy:port"
    }

    stages {
        stage('Network Pre-Checks') {
            steps {
                script {
                    // 1. Verify Docker is installed
                    bat 'docker --version'
                    
                    // 2. Check DNS resolution
                    bat 'nslookup registry-1.docker.io'
                    
                    // 3. Test HTTPS connectivity to Docker Hub
                    def curlStatus = bat(
                        script: 'curl -I https://registry-1.docker.io/v2/ -m 10 --retry 2',
                        returnStatus: true
                    )
                    
                    if (curlStatus != 0) {
                        error("""
                            ❌ Network connectivity test failed!
                            Possible issues:
                            1. No internet access from Jenkins server
                            2. Corporate firewall blocking Docker Hub
                            3. Proxy configuration needed (uncomment http_proxy vars)
                            """)
                    }
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    // Build using docker-compose
                    bat 'docker-compose build'
                    
                    // Tag with build number
                    bat """
                        docker tag ${COMPOSE_PROJECT_NAME}-frontend ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend:${BUILD_NUMBER}
                        docker tag ${COMPOSE_PROJECT_NAME}-backend ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Docker Hub Login') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_CREDENTIALS,
                        passwordVariable: 'DOCKER_TOKEN',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        // Method 1: Standard login
                        def loginStatus = bat(
                            script: """
                                docker logout
                                echo %DOCKER_TOKEN% | docker login -u %DOCKER_USERNAME% --password-stdin
                            """,
                            returnStatus: true
                        )
                        
                        // Method 2: Fallback if first fails
                        if (loginStatus != 0) {
                            echo "⚠️ Standard login failed, trying alternative method"
                            bat """
                                docker logout
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
                                docker login -u %DOCKER_USERNAME% --password-stdin <<< %DOCKER_TOKEN%
                            """
                        }
                        
                        // Final verification
                        def verifyLogin = bat(
                            script: 'docker pull hello-world',
                            returnStatus: true
                        )
                        if (verifyLogin != 0) {
                            error("❌ Docker login verification failed! Check credentials.")
                        }
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

    post {
        always {
            script {
                // Cleanup Docker artifacts
                bat 'docker logout || true'
                bat """
                    docker rmi ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend:${BUILD_NUMBER} || true
                    docker rmi ${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend:${BUILD_NUMBER} || true
                    docker rmi ${COMPOSE_PROJECT_NAME}-frontend || true
                    docker rmi ${COMPOSE_PROJECT_NAME}-backend || true
                """
                cleanWs()
            }
        }
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check network connectivity and Docker Hub credentials.'
            // Optional: Add Slack/email notification here
        }
    }
}