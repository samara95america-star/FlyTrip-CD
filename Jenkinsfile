pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-cd'
        EKS_CLUSTER_NAME = 'fly-eks'

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        stage('1. Checkout FlyTrip CD Repository') {
            steps {
                checkout scm
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - VERIFYING TOOLS"
                    echo "=========================================="

                    echo "Checking AWS CLI..."
                    aws --version

                    echo "Checking AWS credentials..."
                    aws sts get-caller-identity

                    echo "Checking kubectl..."
                    kubectl version --client

                    echo "Checking deployment files..."
                    ls -la

                    echo "Checking Kubernetes Deployment name..."
                    grep -n "name:" deployment.yaml || true
                '''
            }
        }

        stage('3. Connect to EKS') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - CONNECTING TO EKS"
                    echo "=========================================="

                    aws eks update-kubeconfig \
                      --region "${AWS_REGION}" \
                      --name "${EKS_CLUSTER_NAME}"

                    echo "Testing Kubernetes connection..."
                    kubectl get nodes
                '''
            }
        }

        stage('4. Deploy FlyTrip Kubernetes Resources') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - DEPLOYING KUBERNETES RESOURCES"
                    echo "=========================================="

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml

                    echo ""
                    echo "Kubernetes resources applied successfully."
                '''
            }
        }

        stage('5. Update FlyTrip Application Image') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - UPDATING APPLICATION IMAGE"
                    echo "=========================================="

                    echo "Deploying image:"
                    echo "${IMAGE}"

                    kubectl set image \
                      deployment/flytrip \
                      flytrip="${IMAGE}"

                    echo ""
                    echo "Application image updated successfully."
                '''
            }
        }

        stage('6. Wait for FlyTrip Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - WAITING FOR DEPLOYMENT"
                    echo "=========================================="

                    kubectl rollout status \
                      deployment/flytrip \
                      --timeout=180s
                '''
            }
        }

        stage('7. Verify FlyTrip Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - VERIFYING DEPLOYMENT"
                    echo "=========================================="

                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment flytrip

                    echo ""
                    echo "========== SERVICE =========="
                    kubectl get service flytrip-service

                    echo ""
                    echo "FlyTrip deployment verified successfully."
                '''
            }
        }
    }

    post {
        success {
            echo '''
==========================================
FLYTRIP CD PASSED
Application deployed successfully to EKS
==========================================
'''
        }

        failure {
            echo '''
==========================================
FLYTRIP DEPLOYMENT FAILED
Check Jenkins console log and Kubernetes events
==========================================
'''
        }
    }
}
