pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/shux360'
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/havenly_frontend"
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/havenly_backend"
        DOCKER_CREDENTIALS = 'dockerhub-cred'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: env.DOCKER_CREDENTIALS,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        ),
                        string(credentialsId: 'jwt-secret', variable: 'JWT_SECRET'),
                        string(credentialsId: 'mongodb-uri', variable: 'MONGODB_URI')
                    ]) {
                        dir('havenly_frontend-main') {
                            bat "docker build -t %FRONTEND_IMAGE%:%BUILD_NUMBER% ."
                        }
                        dir('havenly_backend-main') {
                            bat """
                                docker build --build-arg MONGODB_URI=%MONGODB_URI% --build-arg JWT_SECRET=%JWT_SECRET% -t %BACKEND_IMAGE%:%BUILD_NUMBER% .
                            """
                        }
                        bat "echo %DOCKER_PASSWORD% | docker login %DOCKER_REGISTRY% -u %DOCKER_USERNAME% --password-stdin"
                        bat "docker push %FRONTEND_IMAGE%:%BUILD_NUMBER%"
                        bat "docker push %BACKEND_IMAGE%:%BUILD_NUMBER%"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                bat """
                    docker rmi %FRONTEND_IMAGE%:%BUILD_NUMBER% || echo "Failed to remove frontend image"
                    docker rmi %BACKEND_IMAGE%:%BUILD_NUMBER% || echo "Failed to remove backend image"
                """
                cleanWs()
            }
        }
    }
}