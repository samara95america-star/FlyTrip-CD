pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-cd'
        EKS_CLUSTER_NAME = 'fly-eks'

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"

        // AWS CLI installed on your Mac via Homebrew
        PATH = "/opt/homebrew/bin:${env.PATH}"
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
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        set -e

                        echo "=========================================="
                        echo "VERIFYING TOOLS"
                        echo "=========================================="

                        echo ""
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
                        echo "Checking Kubernetes manifests..."
                        ls -la

                        echo ""
                        echo "=========================================="
                        echo "TOOLS OK"
                        echo "=========================================="
                    '''
                }
            }
        }

        stage('3. Connect to EKS') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        set -e

                        echo "=========================================="
                        echo "CONNECTING TO EKS"
                        echo "=========================================="

                        aws sts get-caller-identity

                        echo ""
                        echo "Updating kubeconfig..."

                        aws eks update-kubeconfig \
                            --region "${AWS_REGION}" \
                            --name "${EKS_CLUSTER_NAME}"

                        echo ""
                        echo "Testing Kubernetes connection..."

                        kubectl get nodes

                        echo ""
                        echo "EKS CONNECTION OK"
                    '''
                }
            }
        }

        stage('4. Deploy Kubernetes Resources') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        set -e

                        echo "=========================================="
                        echo "DEPLOYING KUBERNETES RESOURCES"
                        echo "=========================================="

                        echo ""
                        echo "Applying deployment.yaml..."
                        kubectl apply -f deployment.yaml

                        echo ""
                        echo "Applying service.yaml..."
                        kubectl apply -f service.yaml

                        echo ""
                        echo "Kubernetes resources deployed."
                    '''
                }
            }
        }

        stage('5. Update Application Image') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        set -e

                        echo "=========================================="
                        echo "UPDATING APPLICATION IMAGE"
                        echo "=========================================="

                        echo "Image:"
                        echo "${IMAGE}"

                        echo ""
                        echo "Updating deployment image..."

                        kubectl set image \
                            deployment/restaurant-company \
                            restaurant-company="${IMAGE}"

                        echo ""
                        echo "Application image updated."
                    '''
                }
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

                    echo ""
                    echo "ROLLOUT COMPLETED"
                '''
            }
        }

        stage('7. Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "VERIFYING DEPLOYMENT"
                    echo "=========================================="

                    echo ""
                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment restaurant-company

                    echo ""
                    echo "========== SERVICE =========="
                    kubectl get service restaurant-company-service

                    echo ""
                    echo "========== APPLICATION IMAGE =========="
                    kubectl get deployment restaurant-company \
                        -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo ""
                    echo ""
                    echo "DEPLOYMENT VERIFICATION COMPLETED"
                '''
            }
        }
    }

    post {
        success {
            echo '''
==========================================
RESTAURANT COMPANY CD PASSED
==========================================

Application deployed successfully to EKS.
'''
        }

        failure {
            echo '''
==========================================
DEPLOYMENT FAILED
==========================================

Check the Jenkins console log.
'''
        }
    }
}
