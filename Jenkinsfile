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
        SSH_OPTS = '-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR'
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

                stage('Prepare EC2 Connection') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                // Use PowerShell for more reliable file operations
                                powershell """
                                    # Create temp directory if it doesn't exist
                                    if (!(Test-Path -Path "${WORKSPACE}\\temp_keys")) {
                                        New-Item -ItemType Directory -Path "${WORKSPACE}\\temp_keys" | Out-Null
                                    }
                                    
                                    # Write the key file
                                    [System.IO.File]::WriteAllText("${WORKSPACE}\\temp_keys\\ec2-key.pem", """${PRIVATE_KEY}""")
                                    
                                    # Set permissions
                                    \$acl = Get-Acl "${WORKSPACE}\\temp_keys\\ec2-key.pem"
                                    \$acl.SetAccessRuleProtection(\$true, \$false) # Disable inheritance
                                    \$acl.Access | ForEach-Object { \$acl.RemoveAccessRule(\$_) } | Out-Null # Remove all rules
                                    
                                    # Add new rules
                                    \$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
                                        "SYSTEM", "FullControl", "Allow"
                                    )
                                    \$acl.AddAccessRule(\$rule)
                                    
                                    \$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
                                        "Administrators", "FullControl", "Allow"
                                    )
                                    \$acl.AddAccessRule(\$rule)
                                    
                                    \$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
                                        [System.Security.Principal.WindowsIdentity]::GetCurrent().Name, 
                                        "Read", 
                                        "Allow"
                                    )
                                    \$acl.AddAccessRule(\$rule)
                                    
                                    Set-Acl "${WORKSPACE}\\temp_keys\\ec2-key.pem" \$acl
                                    
                                    # Make file read-only
                                    Set-ItemProperty -Path "${WORKSPACE}\\temp_keys\\ec2-key.pem" -Name IsReadOnly -Value \$true
                                    
                                    # Verify
                                    Get-Acl "${WORKSPACE}\\temp_keys\\ec2-key.pem" | Format-List
                                """
                            }
                        }
                    }
                }

                stage('Install Docker on EC2') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'ec2-cred', 
                                keyFileVariable: 'PRIVATE_KEY',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                // Create a temporary script file with the commands
                                def dockerInstallScript = """
                                    #!/bin/bash
                                    sudo yum update -y
                                    sudo amazon-linux-extras install docker -y
                                    sudo yum install -y docker
                                    sudo usermod -aG docker ${SSH_USER}
                                    sudo systemctl enable docker
                                    sudo systemctl start docker
                                    sudo chmod 666 /var/run/docker.sock
                                    docker --version
                                """.stripIndent()
                                
                                writeFile file: "${WORKSPACE}/install_docker.sh", text: dockerInstallScript
                                
                                bat """
                                    pscp -i "${WORKSPACE}\\ec2-key.pem" ${SSH_OPTS} "${WORKSPACE}\\install_docker.sh" ${SSH_USER}@${EC2_IP}:/tmp/install_docker.sh
                                    plink -batch -ssh -i "${WORKSPACE}\\ec2-key.pem" ${SSH_USER}@${EC2_IP} ${SSH_OPTS} ^
                                        "chmod +x /tmp/install_docker.sh && /tmp/install_docker.sh && rm -f /tmp/install_docker.sh"
                                    del "${WORKSPACE}\\install_docker.sh"
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
                                    plink -batch -ssh -i "${WORKSPACE}\\ec2-key.pem" ${SSH_USER}@${EC2_IP} ${SSH_OPTS} <<EOF
                                    docker logout
                                    echo ${DOCKER_TOKEN} | docker login -u ${DOCKERHUB_USERNAME} --password-stdin
                                    docker stop ${COMPOSE_PROJECT_NAME}-frontend || true
                                    docker stop ${COMPOSE_PROJECT_NAME}-backend || true
                                    docker rm ${COMPOSE_PROJECT_NAME}-frontend || true
                                    docker rm ${COMPOSE_PROJECT_NAME}-backend || true
                                    docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                                    docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER}
                                    docker run -d --name ${COMPOSE_PROJECT_NAME}-frontend -p 80:5173 -e HOST=0.0.0.0 ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                                    docker run -d --name ${COMPOSE_PROJECT_NAME}-backend -p 3001:3001 -e HOST=0.0.0.0 -e MONGO_URL='${MONGO_URL_SECRET}' -e JWT_SECRET='${JWT_SECRET_SECRET}' ${BACKEND_IMAGE}:${BUILD_NUMBER}
                                    docker ps
                                    EOF
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
                                    plink -batch -ssh -i "${WORKSPACE}\\ec2-key.pem" ${SSH_USER}@${EC2_IP} ${SSH_OPTS} <<EOF
                                    echo "Checking running containers..."
                                    docker ps
                                    echo "Checking frontend logs..."
                                    docker logs ${COMPOSE_PROJECT_NAME}-frontend --tail 50
                                    echo "Checking backend logs..."
                                    docker logs ${COMPOSE_PROJECT_NAME}-backend --tail 50
                                    echo "Checking network connectivity..."
                                    curl -I http://localhost:5173 || true
                                    curl -I http://localhost:3001 || true
                                    EOF
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
            script {
                // Clean up the private key file
                bat """
                    if exist "${WORKSPACE}\\ec2-key.pem" (
                        del "${WORKSPACE}\\ec2-key.pem"
                    )
                """
                bat 'docker logout'
            }
        }
        success {
            echo "Deployment completed successfully! Application should be available at: http://${EC2_IP}"
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}