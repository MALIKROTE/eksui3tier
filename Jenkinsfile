pipeline {
    agent any
    environment {
        EKS_CLUSTER_NAME = "my-eks-cluster"
        AWS_REGION = "ap-south-1"
        DOCKER_HUB_REPO = 'malikdrote'
        DOCKER_HUB_CREDENTIALS = 'docker-hub-credentials'
    }
    stages {
        stage('Install AWS CLI and Kubectl') {
            steps {
                script {
                    sh '''
                        # Check if AWS CLI is installed
                        if ! command -v aws &> /dev/null
                        then
                            echo "AWS CLI not found, installing..."
                            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                            unzip -o awscliv2.zip
                            sudo ./aws/install --update
                        else
                            echo "AWS CLI found, updating..."
                            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                            unzip -o awscliv2.zip
                            sudo ./aws/install --update
                        fi
                        
                        # Install kubectl
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        sudo mv kubectl /usr/local/bin/
                    '''
                }
            }
        }
        stage('Configure AWS CLI and Update Kubeconfig') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh '''
                            aws --version
                            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                            aws configure set default.region $AWS_REGION
                            
                            aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                        '''
                    }
                }
            }
        }
        stage('Build and Push Docker Images') {
            steps {
                script {
                    dir('frontend') {
                        docker.build("${DOCKER_HUB_REPO}/frontend:latest").push('latest')
                    }
                    dir('backend') {
                        docker.build("${DOCKER_HUB_REPO}/backend:latest").push('latest')
                    }
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    sh '''
                        kubectl apply -f k8s/mysql-deployment.yml
                        kubectl apply -f k8s/frontend-deployment.yml
                        kubectl apply -f k8s/backend-deployment.yml
                        kubectl apply -f k8s/frontend-service.yml
                        kubectl apply -f k8s/backend-service.yml
                        kubectl apply -f k8s/mysql-service.yml

                        kubectl get all

                        kubectl get services
                    '''
                }
            }
        }
        stage('Access Application') {
            steps {
                script {
                    def frontendUrl = sh(script: 'kubectl get service frontend-service -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"', returnStdout: true).trim()
                    def backendUrl = sh(script: 'kubectl get service backend-service -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"', returnStdout: true).trim()

                    echo "Frontend URL: http://${frontendUrl}"
                    echo "Backend URL: http://${backendUrl}"
                }
            }
        }
    }
}
