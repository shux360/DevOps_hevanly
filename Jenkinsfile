pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKERHUB_USERNAME = 'shux360'
        COMPOSE_PROJECT_NAME = 'devops_hevanly'
        FRONTEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend"
        BACKEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend"
        EC2_IP = '13.218.71.125'
        EC2_USER = 'ec2-user'
        AWS_REGION = 'us-east-1'
    }

    stages {
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

        stage('EC2 Deployment') {
            stages {
                stage('Configure AWS CLI') {
                    steps {
                        script {
                            withCredentials([[
                                $class: 'AmazonWebServicesCredentialsBinding',
                                credentialsId: 'wsl-ec2',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            ]]) {
                                bat """
                                    aws configure set aws_access_key_id %AWS_ACCESS_KEY_ID%
                                    aws configure set aws_secret_access_key %AWS_SECRET_ACCESS_KEY%
                                    aws configure set region %AWS_REGION%
                                """
                            }
                        }
                    }
                }

                stage('Connect to EC2') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'wsl-ec2', 
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    # Debug: Show key file location and permissions
                                    echo "Private key path: ${PRIVATE_KEY_PATH}"
                                    ls -l ${PRIVATE_KEY_PATH}
                                    
                                    # Set strict permissions for the key
                                    chmod 600 ${PRIVATE_KEY_PATH}
                                    
                                    # Test basic connectivity
                                    echo "Testing connection to port 22..."
                                    nc -zv 13.218.71.125 22
                                    
                                    # Verbose SSH connection test
                                    echo "Attempting SSH connection with verbose output..."
                                    ssh -vvv -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
                                        -i "${PRIVATE_KEY_PATH}" ec2-user@13.218.71.125 \
                                        "echo 'Logged into EC2 successfully!'; \
                                        echo 'EC2 OS Info:'; cat /etc/os-release; \
                                        echo 'Docker status:'; sudo systemctl status docker || echo 'Docker not installed'"
                                """
                            }
                        }
                    }
                }

                stage('Install Docker') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'aws-cred',
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    setlocal
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_ec2_key.pem
                                    copy "%PRIVATE_KEY_PATH%" "%TEMP_KEY%" > nul
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "sudo yum update -y && ^
                                         sudo amazon-linux-extras install docker -y && ^
                                         sudo yum install -y docker && ^
                                         sudo usermod -aG docker %SSH_USER% && ^
                                         sudo systemctl enable docker && ^
                                         sudo systemctl start docker && ^
                                         sudo chmod 666 /var/run/docker.sock && ^
                                         docker --version"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                    endlocal
                                """
                            }
                        }
                    }
                }

                stage('Deploy Containers') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'aws-cred',
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    setlocal
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_ec2_key.pem
                                    copy "%PRIVATE_KEY_PATH%" "%TEMP_KEY%" > nul
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "docker logout && ^
                                         docker login -u %DOCKERHUB_USERNAME% -p %DOCKER_TOKEN% && ^
                                         docker stop ${COMPOSE_PROJECT_NAME}-frontend || true && ^
                                         docker stop ${COMPOSE_PROJECT_NAME}-backend || true && ^
                                         docker rm ${COMPOSE_PROJECT_NAME}-frontend || true && ^
                                         docker rm ${COMPOSE_PROJECT_NAME}-backend || true && ^
                                         docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                         docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                         docker run -d --name ${COMPOSE_PROJECT_NAME}-frontend -p 5173:5173 ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                         docker run -d --name ${COMPOSE_PROJECT_NAME}-backend -p 3001:3001 -e MONGO_URL=${MONGO_URL} -e JWT_SECRET=${JWT_SECRET} ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                         docker ps"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                    endlocal
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