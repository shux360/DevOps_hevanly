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
        EC2_IP = '13.218.71.125'
        EC2_USER = 'ec2-user'
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

//         stage('Deploy') {
//             steps {
//                 bat """
//                     docker-compose down
//                     docker-compose pull
//                     docker-compose up -d
//                 """
//             }
//         }
//     }
// }
 // EC2 Deployment Stages
        stage('EC2 Deployment') {
            stages {
                stage('Connect to EC2') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                // Windows-specific permission fix
                                bat """
                                    icacls "%PRIVATE_KEY_PATH%" /inheritance:r
                                    icacls "%PRIVATE_KEY_PATH%" /grant:r "%USERNAME%":"(R)"
                                    ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "echo 'Logged into EC2 successfully!'"
                                """
                            }
                        }
                    }
                }
                
                stage('Configure Docker Environment') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                bat """
                                ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "
                                sudo usermod -aG docker %EC2_USER%
                                sudo service docker restart
                                docker ps
                                "
                                """
                            }
                        }
                    }
                }
                
                stage('Clean Previous Deployments') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                bat """
                                ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "
                                docker stop ${COMPOSE_PROJECT_NAME}-frontend || true
                                docker stop ${COMPOSE_PROJECT_NAME}-backend || true
                                docker container prune -f
                                docker image prune -a -f
                                "
                                """
                            }
                        }
                    }
                }
                
                stage('Pull New Images') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                bat """
                                ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "
                                docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                                docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER}
                                "
                                """
                            }
                        }
                    }
                }
                
                stage('Deploy Containers') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                // Adjust ports and container configuration as needed
                                bat """
                                ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "
                                docker run -d --name ${COMPOSE_PROJECT_NAME}-frontend -p 5173:5173 ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                                docker run -d --name ${COMPOSE_PROJECT_NAME}-backend -p 3001:3001 ${BACKEND_IMAGE}:${BUILD_NUMBER}
                                "
                                """
                            }
                        }
                    }
                }
                
                stage('Verify Deployment') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                bat """
                                ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "
                                docker ps
                                docker logs ${COMPOSE_PROJECT_NAME}-frontend --tail 50
                                docker logs ${COMPOSE_PROJECT_NAME}-backend --tail 50
                                "
                                """
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            bat 'docker logout'
        }
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}