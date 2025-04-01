pipeline {
    agent any
    
    environment {
        // Docker configuration
        DOCKER_REGISTRY = 'docker.io'
        DOCKERHUB_USERNAME = 'shux360'
        COMPOSE_PROJECT_NAME = 'devops_hevanly'
        FRONTEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-frontend"
        BACKEND_IMAGE = "${DOCKERHUB_USERNAME}/${COMPOSE_PROJECT_NAME}-backend"
        
        // EC2 configuration
        EC2_IP = '13.218.71.125'
        EC2_USER = 'ec2-user'  // Default AWS EC2 user
        
        // AWS Region
        AWS_REGION = 'us-east-1'  // Update with your region
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
                        docker login -u %DOCKER_USERNAME% -p %DOCKER_TOKEN% ${DOCKER_REGISTRY}
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
                                    
                                    :: Test basic connection
                                    ssh -o StrictHostKeyChecking=no -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "echo 'EC2 connection successful'"
                                    
                                    if errorlevel 1 (
                                        echo ERROR: Failed to connect to EC2 instance
                                        exit /b 1
                                    )
                                    
                                    endlocal
                                """
                            }
                        }
                    }
                }
                
                stage('Install Docker on EC2') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'aws-cred',
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% << 'EOF'
                                    #!/bin/bash
                                    sudo yum update -y
                                    sudo amazon-linux-extras install docker -y
                                    sudo yum install -y docker
                                    sudo usermod -aG docker $USER
                                    sudo systemctl enable docker
                                    sudo systemctl start docker
                                    sudo chmod 666 /var/run/docker.sock
                                    docker --version
                                    EOF
                                """
                            }
                        }
                    }
                }
                
                stage('Clean Previous Deployment') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'aws-cred',
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% << 'EOF'
                                    #!/bin/bash
                                    docker stop ${COMPOSE_PROJECT_NAME}-frontend || true
                                    docker stop ${COMPOSE_PROJECT_NAME}-backend || true
                                    docker rm ${COMPOSE_PROJECT_NAME}-frontend || true
                                    docker rm ${COMPOSE_PROJECT_NAME}-backend || true
                                    docker image prune -a -f
                                    EOF
                                """
                            }
                        }
                    }
                }
                
                stage('Login to Docker Hub on EC2') {
                    steps {
                        script {
                            withCredentials([
                                usernamePassword(
                                    credentialsId: 'dockerhub-cred',
                                    passwordVariable: 'DOCKER_TOKEN',
                                    usernameVariable: 'DOCKER_USERNAME'
                                ),
                                sshUserPrivateKey(
                                    credentialsId: 'aws-cred',
                                    keyFileVariable: 'PRIVATE_KEY_PATH',
                                    usernameVariable: 'SSH_USER'
                                )
                            ]) {
                                bat """
                                    ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% << 'EOF'
                                    #!/bin/bash
                                    docker logout
                                    docker login -u ${DOCKER_USERNAME} -p ${DOCKER_TOKEN}
                                    EOF
                                """
                            }
                        }
                    }
                }
                
                stage('Pull New Images on EC2') {
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(
                                credentialsId: 'aws-cred',
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% << 'EOF'
                                    #!/bin/bash
                                    docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                                    docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER}
                                    EOF
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
                                    ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% << 'EOF'
                                    #!/bin/bash
                                    docker run -d \
                                        --name ${COMPOSE_PROJECT_NAME}-frontend \
                                        -p 5173:5173 \
                                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                                    
                                    docker run -d \
                                        --name ${COMPOSE_PROJECT_NAME}-backend \
                                        -p 3001:3001 \
                                        -e MONGO_URL=${MONGO_URL} \
                                        -e JWT_SECRET=${JWT_SECRET} \
                                        ${BACKEND_IMAGE}:${BUILD_NUMBER}
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
                                credentialsId: 'aws-cred',
                                keyFileVariable: 'PRIVATE_KEY_PATH',
                                usernameVariable: 'SSH_USER'
                            )]) {
                                bat """
                                    ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% << 'EOF'
                                    #!/bin/bash
                                    echo "Running containers:"
                                    docker ps
                                    echo -e "\nFrontend logs:"
                                    docker logs ${COMPOSE_PROJECT_NAME}-frontend --tail 50
                                    echo -e "\nBackend logs:"
                                    docker logs ${COMPOSE_PROJECT_NAME}-backend --tail 50
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
            bat 'docker logout'
            script {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'aws-cred',
                    keyFileVariable: 'PRIVATE_KEY_PATH',
                    usernameVariable: 'SSH_USER'
                )]) {
                    bat """
                        ssh -i "%PRIVATE_KEY_PATH%" %SSH_USER%@%EC2_IP% "docker logout"
                    """
                }
            }
        }
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}