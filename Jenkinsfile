pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'

        // CHANGE THESE
        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-cd'
        EKS_CLUSTER_NAME = 'fly-eks'

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        stage('1. Checkout CD Repository') {
            steps {
                checkout scm
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    echo "Checking AWS CLI..."
                    aws --version

                    echo "Checking kubectl..."
                    kubectl version --client

                    echo "Checking deployment files..."
                    ls -la
                '''
            }
        }

        stage('3. Connect to EKS') {
            steps {
                sh '''
                    echo "Connecting Jenkins to EKS..."

                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER_NAME}

                    echo "Testing Kubernetes connection..."
                    kubectl get nodes
                '''
            }
        }

        stage('4. Deploy Kubernetes Resources') {
            steps {
                sh '''
                    echo "Creating/updating Kubernetes resources..."

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml
                '''
            }
        }

        stage('5. Update Application Image') {
            steps {
                sh '''
                    echo "Deploying image:"
                    echo ${IMAGE}

                    kubectl set image \
                      deployment/restaurant-company \
                      restaurant-company=${IMAGE}
                '''
            }
        }

        stage('6. Wait for Deployment') {
            steps {
                sh '''
                    kubectl rollout status \
                      deployment/restaurant-company \
                      --timeout=180s
                '''
            }
        }

        stage('7. Verify Deployment') {
            steps {
                sh '''
                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment restaurant-company

                    echo "========== SERVICE =========="
                    kubectl get service restaurant-company-service
                '''
            }
        }
    }

    post {
        success {
            echo '''
==========================================
RESTAURANT COMPANY CD PASSED
Application deployed successfully to EKS
==========================================
'''
        }

        failure {
            echo '''
==========================================
DEPLOYMENT FAILED
Check Jenkins logs and Kubernetes events
==========================================
'''
        }
    }
}
