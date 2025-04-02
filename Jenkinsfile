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
                                    aws configure set region %AWS_REGION%
                                """
                            }
                        }
                    }
                }

                stage('Verify EC2 Instance') {
                    steps {
                        script {
                            withCredentials([[
                                $class: 'AmazonWebServicesCredentialsBinding',
                                credentialsId: 'aws-cred',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            ]]) {
                                bat """
                                    @echo off
                                    echo "==== VERIFYING EC2 INSTANCE EXISTENCE ===="
                                    aws ec2 describe-instances --region %AWS_REGION% --filters "Name=ip-address,Values=%EC2_IP%" --query "Reservations[*].Instances[*].{InstanceId:InstanceId,State:State.Name,Type:InstanceType}" --output table
                                    
                                    echo "==== CHECKING INSTANCE CONNECTIVITY ===="
                                    ping -n 4 %EC2_IP%
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
                                    @echo off
                                    echo "==== CONFIGURING SECURITY GROUPS ===="
                                    
                                    echo "Finding EC2 instance security groups..."
                                    for /f "tokens=*" %%i in ('aws ec2 describe-instances --region %AWS_REGION% --filters "Name=ip-address,Values=%EC2_IP%" --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text') do set SG_ID=%%i
                                    echo "Security Group ID: %SG_ID%"
                                    
                                    rem Display current security group rules
                                    echo "Current security group rules:"
                                    aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[*].{FromPort:FromPort,ToPort:ToPort,IpProtocol:IpProtocol,IpRanges:IpRanges[*].CidrIp}" --output table
                                    
                                    rem Check and open port 22 (SSH)
                                    for /f "tokens=*" %%p in ('aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[?ToPort==`22`].ToPort" --output text') do set PORT_EXISTS=%%p
                                    if "%PORT_EXISTS%" == "22" (
                                        echo "Port 22 already exists in the security group"
                                    ) else (
                                        echo "Adding port 22 to security group %SG_ID%"
                                        aws ec2 authorize-security-group-ingress --region %AWS_REGION% --group-id %SG_ID% --protocol tcp --port 22 --cidr 0.0.0.0/0
                                    )
                                    
                                    rem Check and open port 3001 (Backend)
                                    for /f "tokens=*" %%p in ('aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[?ToPort==`3001`].ToPort" --output text') do set PORT_EXISTS=%%p
                                    if "%PORT_EXISTS%" == "3001" (
                                        echo "Port 3001 already exists in the security group"
                                    ) else (
                                        echo "Adding port 3001 to security group %SG_ID%"
                                        aws ec2 authorize-security-group-ingress --region %AWS_REGION% --group-id %SG_ID% --protocol tcp --port 3001 --cidr 0.0.0.0/0
                                    )
                                    
                                    rem Check and open port 5173 (Frontend)
                                    for /f "tokens=*" %%p in ('aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[?ToPort==`5173`].ToPort" --output text') do set PORT_EXISTS=%%p
                                    if "%PORT_EXISTS%" == "5173" (
                                        echo "Port 5173 already exists in the security group"
                                    ) else (
                                        echo "Adding port 5173 to security group %SG_ID%"
                                        aws ec2 authorize-security-group-ingress --region %AWS_REGION% --group-id %SG_ID% --protocol tcp --port 5173 --cidr 0.0.0.0/0
                                    )
                                    
                                    rem Verify security group configuration after changes
                                    echo "Updated security group rules:"
                                    aws ec2 describe-security-groups --region %AWS_REGION% --group-ids %SG_ID% --query "SecurityGroups[0].IpPermissions[*].{FromPort:FromPort,ToPort:ToPort,IpProtocol:IpProtocol,IpRanges:IpRanges[*].CidrIp}" --output table
                                """
                            }
                        }
                    }
                }

                stage('Test SSH Connection') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    setlocal EnableDelayedExpansion
                                    echo [INFO] ==== TESTING SSH CONNECTION ====
                                    
                                    :: 1. Create temporary key file
                                    set TEMP_KEY="%WORKSPACE%\\ec2_temp_key.pem"
                                    echo %PRIVATE_KEY% > !TEMP_KEY!
                                    
                                    :: 2. Remove all permissions and set strict access
                                    echo [INFO] Applying strict permissions...
                                    icacls !TEMP_KEY! /inheritance:r /remove:g Everyone /remove:g Users /remove:g "Authenticated Users"
                                    
                                    :: 3. Grant minimal required permissions
                                    for /f "tokens=2 delims=," %%A in ('whoami /user /fo csv ^| findstr /r "\".*\""') do (
                                        set CURRENT_SID=%%~A
                                    )
                                    icacls !TEMP_KEY! /grant *!CURRENT_SID!:(RX)  
                                    icacls !TEMP_KEY! /grant *S-1-5-18:(RX)      
                                    attrib +R !TEMP_KEY!
                                    
                                    :: 4. Verify permissions (debug)
                                    echo [DEBUG] Final permissions:
                                    icacls !TEMP_KEY!
                                    
                                    :: 5. Cache host key first
                                    echo [INFO] Initializing host key...
                                    echo y | plink -ssh -i !TEMP_KEY! -no-antispoof %SSH_USER%@%EC2_IP% exit
                                    
                                    :: 6. Test connection with full debug
                                    echo [INFO] Attempting SSH connection...
                                    plink -v -v -v -batch -ssh -i !TEMP_KEY! %SSH_USER%@%EC2_IP% "echo CONNECTION_SUCCESS && whoami && pwd" > "%WORKSPACE%\\ssh_debug.log" 2>&1
                                    
                                    :: 7. Check results
                                    if !ERRORLEVEL! equ 0 (
                                        echo [SUCCESS] SSH connection verified
                                        type "%WORKSPACE%\\ssh_debug.log" | findstr /i "CONNECTION_SUCCESS"
                                    ) else (
                                        echo [ERROR] SSH failed (Code: !ERRORLEVEL!)
                                        echo [DEBUG] Last 10 lines of log:
                                        tail -10 "%WORKSPACE%\\ssh_debug.log"
                                        exit /b 1
                                    )
                                    
                                    :: 8. Secure cleanup
                                    cipher /w:!TEMP_KEY! >nul 2>&1
                                    del !TEMP_KEY! >nul 2>&1
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
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    echo "==== INSTALLING DOCKER ===="
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_ec2_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo 'Checking if Docker is already installed...' && ^
                                         if command -v docker &> /dev/null; then ^
                                           echo 'Docker is already installed. Version:' && ^
                                           docker --version; ^
                                         else ^
                                           echo 'Installing Docker...' && ^
                                           sudo yum update -y && ^
                                           sudo amazon-linux-extras install docker -y && ^
                                           sudo yum install -y docker && ^
                                           sudo service docker start && ^
                                           sudo systemctl enable docker && ^
                                           sudo usermod -a -G docker %SSH_USER% && ^
                                           sudo chmod 666 /var/run/docker.sock && ^
                                           echo 'Docker installation complete. Version:' && ^
                                           docker --version || echo 'DOCKER INSTALLATION FAILED'; ^
                                         fi && ^
                                         echo 'Installing additional utilities...' && ^
                                         sudo yum install -y nc"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
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
                                    echo "==== DEPLOYING CONTAINERS ===="
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_ec2_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo 'Logging into Docker Hub...' && ^
                                        docker logout && ^
                                        docker login -u %DOCKERHUB_USERNAME% -p %DOCKER_TOKEN% && ^
                                        echo 'Stopping and removing existing containers...' && ^
                                        docker stop ${COMPOSE_PROJECT_NAME}-frontend ${COMPOSE_PROJECT_NAME}-backend 2>/dev/null || true && ^
                                        docker rm ${COMPOSE_PROJECT_NAME}-frontend ${COMPOSE_PROJECT_NAME}-backend 2>/dev/null || true && ^
                                        echo 'Creating docker network...' && ^
                                        docker network create app-network 2>/dev/null || true && ^
                                        echo 'Pulling latest images...' && ^
                                        docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                        docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo 'Starting backend container...' && ^
                                        docker run -d --name ${COMPOSE_PROJECT_NAME}-backend --network app-network -p 0.0.0.0:3001:3001 -e HOST=0.0.0.0 -e PORT=3001 -e MONGO_URL='%MONGO_URL_SECRET%' -e JWT_SECRET='%JWT_SECRET_SECRET%' ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo 'Waiting for backend to start...' && ^
                                        sleep 5 && ^
                                        echo 'Starting frontend container...' && ^
                                        docker run -d --name ${COMPOSE_PROJECT_NAME}-frontend --network app-network -p 0.0.0.0:5173:5173 -e HOST=0.0.0.0 -e PORT=5173 -e API_URL=http://%EC2_IP%:3001 ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo 'Containers started. Current running containers:' && ^
                                        docker ps"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                """
                            }
                        }
                    }
                }
                
                stage('Initial Verification') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    echo "==== INITIAL VERIFICATION ===="
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_verify_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    echo "Waiting 15 seconds for containers to fully start up..."
                                    timeout /t 15 /nobreak
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo '==== Checking running containers ====' && ^
                                        docker ps -a && ^
                                        echo '==== Checking container status ====' && ^
                                        docker inspect -f '{{.State.Status}} - {{.State.Running}}' ${COMPOSE_PROJECT_NAME}-frontend && ^
                                        docker inspect -f '{{.State.Status}} - {{.State.Running}}' ${COMPOSE_PROJECT_NAME}-backend && ^
                                        echo '==== Checking frontend logs ====' && ^
                                        docker logs ${COMPOSE_PROJECT_NAME}-frontend --tail 20 && ^
                                        echo '==== Checking backend logs ====' && ^
                                        docker logs ${COMPOSE_PROJECT_NAME}-backend --tail 20"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                """
                            }
                        }
                    }
                }
                
                stage('Enhanced Diagnostics') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    echo "==== ENHANCED DIAGNOSTICS ===="
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_verify_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo '1. Checking network configuration:' && ^
                                        echo '- Docker network:' && ^
                                        docker network inspect app-network && ^
                                        echo '- Container network settings:' && ^
                                        docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} - {{.NetworkSettings.Ports}}' ${COMPOSE_PROJECT_NAME}-frontend && ^
                                        docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} - {{.NetworkSettings.Ports}}' ${COMPOSE_PROJECT_NAME}-backend && ^
                                        echo '2. Checking port bindings on host:' && ^
                                        sudo netstat -tulpn | grep -E '5173|3001' && ^
                                        echo '3. Check host firewall:' && ^
                                        sudo iptables -L | grep -E '5173|3001|DROP|REJECT' && ^
                                        echo '4. Testing network connectivity:' && ^
                                        echo '- From container to host:' && ^
                                        docker exec ${COMPOSE_PROJECT_NAME}-frontend curl -v localhost:5173 || echo 'Frontend curl failed' && ^
                                        docker exec ${COMPOSE_PROJECT_NAME}-backend curl -v localhost:3001 || echo 'Backend curl failed' && ^
                                        echo '- From host to container:' && ^
                                        curl -v http://localhost:5173 || echo 'Host to frontend curl failed' && ^
                                        curl -v http://localhost:3001 || echo 'Host to backend curl failed' && ^
                                        echo '5. Environment variables:' && ^
                                        docker exec ${COMPOSE_PROJECT_NAME}-frontend env | grep -E 'HOST|PORT|API' && ^
                                        docker exec ${COMPOSE_PROJECT_NAME}-backend env | grep -E 'HOST|PORT'"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                """
                            }
                        }
                    }
                }
                
                stage('Fix Common Issues') {
                    steps {
                        script {
                            withCredentials([
                                sshUserPrivateKey(
                                    credentialsId: 'ec2-cred', 
                                    keyFileVariable: 'PRIVATE_KEY',
                                    usernameVariable: 'SSH_USER'
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
                                    echo "==== APPLYING FIXES ===="
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_fix_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo 'Applying common fixes...' && ^
                                        echo '1. Ensuring containers use 0.0.0.0 binding...' && ^
                                        docker stop ${COMPOSE_PROJECT_NAME}-frontend ${COMPOSE_PROJECT_NAME}-backend && ^
                                        docker rm ${COMPOSE_PROJECT_NAME}-frontend ${COMPOSE_PROJECT_NAME}-backend && ^
                                        echo '2. Restarting containers with explicit bind address...' && ^
                                        docker run -d --name ${COMPOSE_PROJECT_NAME}-backend --network app-network -p 0.0.0.0:3001:3001 -e HOST=0.0.0.0 -e PORT=3001 -e MONGO_URL='%MONGO_URL_SECRET%' -e JWT_SECRET='%JWT_SECRET_SECRET%' ${BACKEND_IMAGE}:${BUILD_NUMBER} && ^
                                        sleep 5 && ^
                                        docker run -d --name ${COMPOSE_PROJECT_NAME}-frontend --network app-network -p 0.0.0.0:5173:5173 -e HOST=0.0.0.0 -e PORT=5173 -e API_URL=http://%EC2_IP%:3001 ${FRONTEND_IMAGE}:${BUILD_NUMBER} && ^
                                        echo '3. Disabling firewall temporarily to check if it is the issue...' && ^
                                        sudo iptables -F || echo 'Could not flush firewall rules' && ^
                                        echo '4. Making sure ports are free...' && ^
                                        sudo lsof -i:5173 -i:3001 || echo 'No other processes using these ports' && ^
                                        echo '5. Current running containers:' && ^
                                        docker ps"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                """
                            }
                        }
                    }
                }
                
                stage('Final Verification') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    @echo off
                                    echo "==== FINAL VERIFICATION ===="
                                    
                                    set TEMP_KEY=%WORKSPACE%\\temp_final_key.pem
                                    echo %PRIVATE_KEY% > "%TEMP_KEY%"
                                    icacls "%TEMP_KEY%" /inheritance:r
                                    icacls "%TEMP_KEY%" /grant:r "%USERNAME%":F
                                    
                                    echo "Waiting 15 seconds for changes to take effect..."
                                    timeout /t 15 /nobreak
                                    
                                    plink -batch -ssh -i "%TEMP_KEY%" %SSH_USER%@%EC2_IP% ^
                                        "echo 'Final verification...' && ^
                                        echo '1. Container status:' && ^
                                        docker ps && ^
                                        echo '2. Port bindings:' && ^
                                        sudo netstat -tulpn | grep -E '5173|3001' && ^
                                        echo '3. Internal connectivity test:' && ^
                                        curl -v http://localhost:5173 || echo 'Frontend still not accessible internally' && ^
                                        curl -v http://localhost:3001 || echo 'Backend still not accessible internally' && ^
                                        echo '4. External IP test:' && ^
                                        curl -v http://%EC2_IP%:5173 || echo 'Frontend not accessible via public IP' && ^
                                        curl -v http://%EC2_IP%:3001 || echo 'Backend not accessible via public IP'"
                                    
                                    del "%TEMP_KEY%" > nul 2>&1
                                """
                            }
                        }
                    }
                }
                
                stage('Test External Access') {
                    steps {
                        script {
                            bat """
                                @echo off
                                echo "==== TESTING EXTERNAL ACCESS ===="
                                
                                echo "Testing if frontend is accessible from Jenkins..."
                                curl -v -m 20 -o frontend_response.txt ${FRONTEND_URL} || echo "Failed to connect to frontend"
                                if exist frontend_response.txt type frontend_response.txt
                                
                                echo "Testing if backend is accessible from Jenkins..."
                                curl -v -m 20 -o backend_response.txt ${BACKEND_URL} || echo "Failed to connect to backend"
                                if exist backend_response.txt type backend_response.txt
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
                @echo off
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
                echo 3. Try accessing the site from a different network or browser
                echo 4. Make sure your application is binding to 0.0.0.0 and not just localhost (127.0.0.1)
                echo 5. Check if the EC2 instance has any additional firewall rules
                echo.
            """
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
            
            // Add diagnostic information
            bat """
                @echo off
                echo.
                echo ===============================================
                echo =    DEPLOYMENT FAILED - TROUBLESHOOTING     =
                echo ===============================================
                echo.
                echo 1. Check if the EC2 instance (${EC2_IP}) is reachable:
                ping -n 3 ${EC2_IP}
                echo.
                echo 2. Verify your AWS and SSH credentials in Jenkins
                echo 3. Check if the security group is correctly configured
                echo 4. Review Docker logs for application errors
                echo 5. Verify that your application is properly configured to listen on 0.0.0.0
                echo.
            """
        }
    }
}