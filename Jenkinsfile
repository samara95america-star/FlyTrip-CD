pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-cd'
        EKS_CLUSTER_NAME = 'fly-eks'

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"

        // Make AWS CLI and kubectl available to Jenkins
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
    }

    stages {

        stage('1. Checkout CD Repository') {
            steps {
                checkout scm
            }
        }

        stage('2. Verify Tools') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                        set -e

                        echo "=========================================="
                        echo "VERIFYING TOOLS"
                        echo "=========================================="

                        echo "Checking AWS CLI..."
                        which aws
                        aws --version

                        echo ""
                        echo "Checking AWS credentials..."
                        aws sts get-caller-identity

                        echo ""
                        echo "Checking kubectl..."
                        which kubectl
                        kubectl version --client

                        echo ""
                        echo "Checking deployment files..."
                        ls -la
                    '''
                }
            }
        }

        stage('3. Connect to EKS') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                        set -e

                        echo "=========================================="
                        echo "CONNECTING TO EKS"
                        echo "=========================================="

                        aws eks update-kubeconfig \
                          --region "$AWS_REGION" \
                          --name "$EKS_CLUSTER_NAME"

                        echo ""
                        echo "Testing Kubernetes connection..."
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('4. Deploy Kubernetes Resources') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "DEPLOYING KUBERNETES RESOURCES"
                    echo "=========================================="

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml
                '''
            }
        }

        stage('5. Update Application Image') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "UPDATING APPLICATION IMAGE"
                    echo "=========================================="

                    echo "Deploying image:"
                    echo "$IMAGE"

                    kubectl set image \
                      deployment/restaurant-company \
                      restaurant-company="$IMAGE"
                '''
            }
        }

        stage('6. Wait for Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "WAITING FOR DEPLOYMENT"
                    echo "=========================================="

                    kubectl rollout status \
                      deployment/restaurant-company \
                      --timeout=180s
                '''
            }
        }

        stage('7. Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment restaurant-company

                    echo ""
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
Check Jenkins console log for the exact error.
==========================================
'''
        }
    }
}
