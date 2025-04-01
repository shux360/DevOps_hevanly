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
                                credentialsId: 'aws-cred', 
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    setlocal
                                    set DEBUG_LOG=%WORKSPACE%\\ssh_connection.log
                                    
                                    :: 1. Verify private key exists
                                    if not exist "%PRIVATE_KEY_PATH%" (
                                        echo ERROR: Private key not found at %PRIVATE_KEY_PATH%
                                        exit /b 1
                                    )
                                    
                                    :: 2. Fix key permissions (Windows specific)
                                    echo Fixing key permissions... >> %DEBUG_LOG%
                                    icacls "%PRIVATE_KEY_PATH%" /inheritance:r >> %DEBUG_LOG% 2>&1
                                    icacls "%PRIVATE_KEY_PATH%" /grant:r "%USERNAME%":F >> %DEBUG_LOG% 2>&1
                                    
                                    :: 3. First try with native SSH (proper case for ConfigFile)
                                    echo Trying native SSH connection... >> %DEBUG_LOG%
                                    ssh -v -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL -o ConfigFile=/dev/null -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'Native SSH success'" >> %DEBUG_LOG% 2>&1
                                    
                                    if not errorlevel 1 (
                                        echo SUCCESS: Connected using native SSH
                                        exit /b 0
                                    )
                                    
                                    :: 4. If native SSH failed, try with Plink and accept host key
                                    echo Native SSH failed, trying Plink... >> %DEBUG_LOG%
                                    if exist "C:\\Program Files\\PuTTY\\plink.exe" (
                                        echo Accepting host key for Plink... >> %DEBUG_LOG%
                                        echo y | "C:\\Program Files\\PuTTY\\plink.exe" -ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% exit >> %DEBUG_LOG% 2>&1
                                        
                                        echo Testing Plink connection... >> %DEBUG_LOG%
                                        "C:\\Program Files\\PuTTY\\plink.exe" -batch -ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'Plink success'" >> %DEBUG_LOG% 2>&1
                                        
                                        if not errorlevel 1 (
                                            echo SUCCESS: Connected using Plink
                                            exit /b 0
                                        )
                                    )
                                    
                                    :: 5. Final fallback - add host key to known hosts manually
                                    echo Trying manual host key acceptance... >> %DEBUG_LOG%
                                    if not exist "%USERPROFILE%\\.ssh" mkdir "%USERPROFILE%\\.ssh" >> %DEBUG_LOG% 2>&1
                                    echo [%EC2_IP%]:22 ssh-ed25519 SHA256:saG9PJi0rIn3RwJMHcnxNqKdWI3ZfXZjIoGK+WbLWA0 >> "%USERPROFILE%\\.ssh\\known_hosts"
                                    
                                    ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'Manual host key success'" >> %DEBUG_LOG% 2>&1
                                    
                                    if errorlevel 1 (
                                        echo ERROR: All connection attempts failed
                                        type %DEBUG_LOG%
                                        exit /b 1
                                    )
                                    
                                    echo SUCCESS: Connected after manual host key setup
                                    endlocal
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