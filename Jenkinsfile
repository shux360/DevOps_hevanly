pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKERHUB_USERNAME = 'shux360'
        COMPOSE_PROJECT_NAME = 'devops_hevanly'
        FRONTEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend"
        BACKEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend"
        EC2_IP = '13.218.71.125'
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
                    usernameVariable: 'DOCKER_USERNAME'
                )]) {
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

        // EC2 Deployment Stages
        stage('EC2 Deployment') {
            stages {
                // stage('Connect to EC2') {
                //     steps {
                //         script {
                //             withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH', usernameVariable: 'SSH_USER')]) {
                //                 // Using bat for Windows permission handling
                //                 bat """
                //                     @echo off
                //                     setlocal
                                    
                //                     :: Fix permissions using icacls
                //                     icacls "%PRIVATE_KEY_PATH%" /inheritance:r
                //                     icacls "%PRIVATE_KEY_PATH%" /grant:r "%SSH_USER%":F
                //                     icacls "%PRIVATE_KEY_PATH%" /remove:g "Everyone" "Authenticated Users" "BUILTIN\\Users"
                                    
                //                     :: Test connection
                //                     ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'EC2 connection successful'"
                                    
                //                     endlocal
                //                 """
                //             }
                //         }
                //     }
                // }
                stage('Connect to EC2') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'wsl-ec2', 
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                // Simplified and more reliable approach
                                bat """
                                    @echo off
                                    setlocal
                                    
                                    :: 1. Use a more reliable SSH command that doesn't require strict permissions
                                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'EC2 connection successful'"
                                    
                                    :: 2. Alternative approach if above fails (no permission changes needed)
                                    if errorlevel 1 (
                                        echo Trying alternative connection method...
                                        :: Use plink if available (from PuTTY)
                                        if exist "C:\\Program Files\\PuTTY\\plink.exe" (
                                            "C:\\Program Files\\PuTTY\\plink.exe" -batch -ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'EC2 connection successful'"
                                        ) else (
                                            echo ERROR: Failed to connect to EC2 instance
                                            exit /b 1
                                        )
                                    )
                                    
                                    endlocal
                                """
                            }
                        }
                    }
                }
                
                stage('Install Docker') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: 'wsl-ec2', keyFileVariable: 'PRIVATE_KEY_PATH')]) {
                                bat """
                                    ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %EC2_USER%@%EC2_IP% "
                                    sudo yum update -y || true
                                    sudo amazon-linux-extras install docker -y || true
                                    sudo yum install -y docker || true
                                    sudo usermod -aG docker %EC2_USER%
                                    sudo systemctl enable docker
                                    sudo systemctl restart docker
                                    "
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
                                    sudo amazon-linux-extras install docker -y || true
                                    sudo yum install -y docker || true
                                    sudo usermod -aG docker %EC2_USER%
                                    sudo systemctl enable docker
                                    sudo systemctl restart docker
                                    sudo chmod 666 /var/run/docker.sock
                                    docker --version
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
                                    echo 'Running containers:'
                                    docker ps
                                    echo '\nFrontend logs:'
                                    docker logs ${COMPOSE_PROJECT_NAME}-frontend --tail 50
                                    echo '\nBackend logs:'
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