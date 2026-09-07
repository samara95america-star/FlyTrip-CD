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

        stage('1. Checkout FlyTrip CD Repository') {
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
                        echo "FLYTRIP CD - VERIFYING TOOLS"
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

        stage('3. Connect to FlyTrip EKS') {
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
                        echo "FLYTRIP - CONNECTING TO EKS"
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
                    echo "$IMAGE"

                    kubectl set image \
                      -f deployment.yaml \
                      "*=$IMAGE"

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
                      -f deployment.yaml \
                      --timeout=180s
                '''
            }
        }

        stage('7. Verify FlyTrip Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - DEPLOYMENT VERIFICATION"
                    echo "=========================================="

                    echo ""
                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo ""
                    echo "========== DEPLOYMENTS =========="
                    kubectl get deployments

                    echo ""
                    echo "========== SERVICES =========="
                    kubectl get services

                    echo ""
                    echo "=========================================="
                    echo "FLYTRIP DEPLOYMENT VERIFIED"
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
Check the Jenkins console log for the exact error.
==========================================
'''
        }
    }
}
