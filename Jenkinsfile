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
        // Monitoring variables
        PROMETHEUS_PORT = '9090'
        GRAFANA_PORT = '3000'
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
                bat '''
                    if not exist monitoring mkdir monitoring
                    if not exist monitoring\\prometheus mkdir monitoring\\prometheus
                    if not exist monitoring\\grafana mkdir monitoring\\grafana
                '''
            }
        }
        stage('Configure Monitoring') {
            steps {
                // Create Prometheus config
                writeFile file: 'monitoring/prometheus/prometheus.yml', text: """
                global:
                  scrape_interval: 15s
                
                scrape_configs:
                  - job_name: 'prometheus'
                    static_configs:
                      - targets: ['localhost:${PROMETHEUS_PORT}']
                  - job_name: 'backend'
                    static_configs:
                      - targets: ['localhost:3001']
                  - job_name: 'frontend'
                    static_configs:
                      - targets: ['localhost:5173']
                """
                
                // Create basic Grafana config
                writeFile file: 'monitoring/grafana/grafana.ini', text: """
                [server]
                http_addr = 0.0.0.0
                http_port = ${GRAFANA_PORT}
                
                [security]
                admin_user = admin
                admin_password = admin
                
                [database]
                type = sqlite3
                """
            }
        }

        stage('Build with Docker Compose') {
            steps {
                bat 'docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml build'
                
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
                    bat """
                        docker logout
                        docker login -u %DOCKER_USERNAME% -p %DOCKER_TOKEN%
                    """
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

        stage('Deploy Application and Monitoring') {
            steps {
                bat """
                    docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml down
                    docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
                """
            }
        }
        
        stage('Verify Monitoring') {
            steps {
                script {
                    // Wait longer for containers to initialize
                    sleep(time: 30, unit: 'SECONDS')
                    
                    // Get container status first
                    bat 'docker ps -a'
                    bat 'docker logs ${COMPOSE_PROJECT_NAME}-prometheus-1'
                    
                    // Improved health check with retries
                    def prometheusReady = false
                    def grafanaReady = false
                    int retries = 5
                    int delay = 10
                    
                    for (int i = 0; i < retries; i++) {
                        try {
                            if (!prometheusReady) {
                                def prometheusStatus = bat(
                                    script: 'curl -s -o nul -w "%%{http_code}" http://localhost:9090/api/v1/targets',
                                    returnStdout: true
                                ).trim()
                                if (prometheusStatus == "200") {
                                    prometheusReady = true
                                    echo "Prometheus is ready (attempt ${i + 1})"
                                }
                            }
                            
                            if (!grafanaReady) {
                                def grafanaStatus = bat(
                                    script: 'curl -s -o nul -w "%%{http_code}" http://localhost:3000/api/health',
                                    returnStdout: true
                                ).trim()
                                if (grafanaStatus == "200") {
                                    grafanaReady = true
                                    echo "Grafana is ready (attempt ${i + 1})"
                                }
                            }
                            
                            if (prometheusReady && grafanaReady) break
                            
                            sleep(time: delay, unit: 'SECONDS')
                        } catch (Exception e) {
                            echo "Attempt ${i + 1} failed: ${e.getMessage()}"
                            sleep(time: delay, unit: 'SECONDS')
                        }
                    }
                    
                    if (!prometheusReady || !grafanaReady) {
                        bat 'docker-compose -f docker-compose.monitoring.yml logs'
                        error("Monitoring services failed to start. Prometheus ready: ${prometheusReady}, Grafana ready: ${grafanaReady}")
                    }
                }
            }
        }
    }
    post {
        always {
            echo """
            Deployment Summary:
            - Application deployed
            - Monitoring available at:
              • Prometheus: http://localhost:${PROMETHEUS_PORT}
              • Grafana: http://localhost:${GRAFANA_PORT} (admin/admin)
            """
        }
    }
}