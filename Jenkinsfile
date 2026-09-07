pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-cd'
        EKS_CLUSTER_NAME = 'fly-eks'

        AWS_CLI = '/opt/homebrew/bin/aws'
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
                    test -x "${AWS_CLI}"
                    "${AWS_CLI}" sts get-caller-identity

                    echo ""
                    echo "Checking kubectl..."
                    command -v kubectl
                    kubectl version --client

                    echo ""
                    echo "Checking Kubernetes files..."
                    test -f deployment.yaml
                    test -f service.yaml

                    echo ""
                    echo "Files found:"
                    ls -la deployment.yaml service.yaml

                    echo ""
                    echo "Tools and files verified successfully."
                '''
            }
        }

        stage('3. Connect to FlyTrip EKS') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - CONNECTING TO EKS"
                    echo "=========================================="

                    "${AWS_CLI}" eks update-kubeconfig \
                        --region "${AWS_REGION}" \
                        --name "${EKS_CLUSTER_NAME}"

                    echo ""
                    echo "Current Kubernetes context:"
                    kubectl config current-context

                    echo ""
                    echo "Testing Kubernetes connection..."
                    kubectl get nodes

                    echo ""
                    echo "Connected to EKS cluster: ${EKS_CLUSTER_NAME}"
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
                    echo "FlyTrip Kubernetes resources applied successfully."
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
                    echo "FlyTrip application image updated successfully."
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

                    echo ""
                    echo "FlyTrip deployment completed successfully."
                '''
            }
        }

        stage('7. Verify FlyTrip Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - FINAL VERIFICATION"
                    echo "=========================================="

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment flytrip

                    echo ""
                    echo "========== PODS =========="
                    kubectl get pods -l app=flytrip -o wide

                    echo ""
                    echo "========== SERVICE =========="
                    kubectl get service flytrip-service

                    echo ""
                    echo "=========================================="
                    echo "FLYTRIP DEPLOYMENT VERIFIED SUCCESSFULLY"
                    echo "=========================================="
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
Check Jenkins console output
==========================================
'''
        }
    }
}
