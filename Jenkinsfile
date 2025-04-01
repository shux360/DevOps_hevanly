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
        // Add public URLs for verification
        FRONTEND_URL = "http://${EC2_IP}:5173"
        BACKEND_URL = "http://${EC2_IP}:3001"
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
                                credentialsId: 'aws-cred',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            ]]) {
                                bat """
                                    aws configure set aws_access_key_id %AWS_ACCESS_KEY_ID%
                                    aws configure set aws_secret_access_key %AWS_SECRET_ACCESS_KEY%
                                    aws configure set region us-east-1
                                """
                            }
                        }
                    }
                }

                stage('Configure Security Groups') {
                    steps {
                        script {
                            withCredentials([[
                                $class: 'AmazonWebServicesCredentialsBinding',
                                credentialsId: 'aws-cred',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            ]]) {
                                bat """
                                    echo "Finding EC2 instance security groups..."
                                    for /f "tokens=*" %%i in ('aws ec2 describe-instances --region %AWS_REGION% --filters "Name=ip-address,Values=%EC2_IP%" --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text') do set SG_ID=%%i
                                    echo "Security Group ID: %SG_ID%"
                                    
                                    :: Check if port 3001 is open using a more reliable method
                                    for /f "tokens=*" %%p in ('aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[?ToPort==`3001`].ToPort" --output text') do set PORT_EXISTS=%%p
                                    
                                    if "%PORT_EXISTS%" == "3001" (
                                        echo "Port 3001 already exists in the security group"
                                    ) else (
                                        echo "Adding port 3001 to security group %SG_ID%"
                                        aws ec2 authorize-security-group-ingress --region %AWS_REGION% --group-id %SG_ID% --protocol tcp --port 3001 --cidr 0.0.0.0/0
                                    )
                                    
                                    :: Repeat for port 5173
                                    for /f "tokens=*" %%p in ('aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[?ToPort==\`5173\`].ToPort" --output text') do set PORT_EXISTS=%%p
                                    
                                    if "%PORT_EXISTS%" == "5173" (
                                        echo "Port 5173 already exists in the security group"
                                    ) else (
                                        echo "Adding port 5173 to security group %SG_ID%"
                                        aws ec2 authorize-security-group-ingress --region %AWS_REGION% --group-id %SG_ID% --protocol tcp --port 5173 --cidr 0.0.0.0/0
                                    )
                                """
                            }
                        }
                    }
                }

                stage('Access EC2 Instance') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    echo %PRIVATE_KEY% > "%TEMP%\\ec2-key.pem"
                                    
                                    :: Fix permissions
                                    icacls "%TEMP%\\ec2-key.pem" /inheritance:r
                                    icacls "%TEMP%\\ec2-key.pem" /grant:r "SYSTEM:(R)"
                                    icacls "%TEMP%\\ec2-key.pem" /grant:r "%USERNAME%:(R)"
                                    
                                    :: Connect with verbose output for debugging
                                    ssh -vvv -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL -i "%TEMP%\\ec2-key.pem" %SSH_USER%@%EC2_IP% "echo Connected successfully && whoami"
                                    
                                    :: Clean up
                                    del "%TEMP%\\ec2-key.pem"
                                """
                            }
                        }
                    }
                }

                stage('Install Docker') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    setlocal
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_ec2_key.pem
                                    copy "%PRIVATE_KEY%" "%TEMP_KEY%" > nul
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo 'Installing Docker...' && ^
                                         sudo yum update -y && ^
                                         sudo amazon-linux-extras install docker -y && ^
                                         sudo yum install -y docker && ^
                                         sudo service docker start && ^
                                         sudo systemctl enable docker && ^
                                         sudo usermod -a -G docker %SSH_USER% && ^
                                         sudo chmod 666 /var/run/docker.sock && ^
                                         echo 'Docker installation complete. Version:' && ^
                                         docker --version || echo 'DOCKER INSTALLATION FAILED' && ^
                                         echo 'Installing additional utilities...' && ^
                                         sudo yum install -y nc"
                                    
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
                            withCredentials([
                                sshUserPrivateKey(
                                    credentialsId: 'ec2-cred', 
                                    keyFileVariable: 'PRIVATE_KEY',
                                    usernameVariable: 'SSH_USER'
                                ),
                                usernamePassword(
                                    credentialsId: 'dockerhub-cred',
                                    usernameVariable: 'DOCKERHUB_USERNAME',
                                    passwordVariable: 'DOCKER_TOKEN'
                                ),
                                string(
                                    credentialsId: 'MONGO_URL',
                                    variable: 'MONGO_URL_SECRET'
                                ),
                                string(
                                    credentialsId: 'JWT_SECRET',
                                    variable: 'JWT_SECRET_SECRET'
                                )
                            ]) {
                                bat """
                                    @echo off
                                    setlocal
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_ec2_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo 'Logging into Docker Hub...' && ^
                                        docker logout && ^
                                        docker login -u %DOCKERHUB_USERNAME% -p %DOCKER_TOKEN% && ^
                                        echo 'Stopping and removing existing containers...' && ^
                                        docker stop ${COMPOSE_PROJECT_NAME}-frontend || true && ^
                                        docker stop ${COMPOSE_PROJECT_NAME}-backend || true && ^
                                        docker rm ${COMPOSE_PROJECT_NAME}-frontend || true && ^
                                        docker rm ${COMPOSE_PROJECT_NAME}-backend || true && ^
                                        echo 'Creating docker network...' && ^
                                        docker network create app-network || true && ^
                                        echo 'Pulling latest images...' && ^
                                        docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                        docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo 'Starting frontend container...' && ^
                                        docker run -d --name ${COMPOSE_PROJECT_NAME}-frontend --network app-network -p 0.0.0.0:5173:5173 -e HOST=0.0.0.0 -e API_URL=http://%EC2_IP%:3001 ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo 'Starting backend container...' && ^
                                        docker run -d --name ${COMPOSE_PROJECT_NAME}-backend --network app-network -p 0.0.0.0:3001:3001 -e HOST=0.0.0.0 -e MONGO_URL='%MONGO_URL_SECRET%' -e JWT_SECRET='%JWT_SECRET_SECRET%' ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo 'Containers started. Current running containers:' && ^
                                        docker ps"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                    endlocal
                                """
                            }
                        }
                    }
                }
                
                stage('Verify Deployment') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    setlocal
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_verify_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    echo "Waiting 30 seconds for containers to fully start up..."
                                    timeout /t 30 /nobreak
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo '==== Checking running containers ====' && ^
                                        docker ps && ^
                                        echo '==== Checking frontend logs ====' && ^
                                        docker logs %COMPOSE_PROJECT_NAME%-frontend --tail 20 && ^
                                        echo '==== Checking backend logs ====' && ^
                                        docker logs %COMPOSE_PROJECT_NAME%-backend --tail 20 && ^
                                        echo '==== Verifying ports are actually listening ====' && ^
                                        sudo netstat -tulpn | grep -E '5173|3001' && ^
                                        echo '==== Testing local connectivity ====' && ^
                                        curl -s -o /dev/null -w '%%{http_code}\\n' http://localhost:5173 && ^
                                        curl -s -o /dev/null -w '%%{http_code}\\n' http://localhost:3001 && ^
                                        echo '==== Checking network configuration inside containers ====' && ^
                                        docker exec %COMPOSE_PROJECT_NAME%-frontend env | grep -E 'HOST|API_URL' && ^
                                        docker exec %COMPOSE_PROJECT_NAME%-backend env | grep HOST"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                    endlocal
                                """
                            }
                        }
                    }
                }
                
                stage('Test External Access') {
                    steps {
                        script {
                            // Test if frontend is accessible from Jenkins server
                            bat """
                                echo "Testing if frontend is accessible from Jenkins..."
                                curl -m 10 -s -o nul -w "%%{http_code}" ${FRONTEND_URL} || echo "Failed to connect to frontend"
                                
                                echo "Testing if backend is accessible from Jenkins..."
                                curl -m 10 -s -o nul -w "%%{http_code}" ${BACKEND_URL} || echo "Failed to connect to backend"
                            """
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
            echo "Your application should be accessible at: ${FRONTEND_URL}"
            echo "Your backend API should be accessible at: ${BACKEND_URL}"
            
            // Add some visual feedback in the Jenkins console
            bat """
                echo.
                echo ===============================================
                echo =    DEPLOYMENT COMPLETED SUCCESSFULLY!      =
                echo ===============================================
                echo.
                echo Frontend: ${FRONTEND_URL}
                echo Backend:  ${BACKEND_URL}
                echo.
                echo If you still have issues accessing the application:
                echo 1. Verify the security group has TCP ports 5173 and 3001 open
                echo 2. Check that your frontend is configured to use the correct backend URL
                echo 3. Try accessing the site from a different network
                echo.
            """
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}