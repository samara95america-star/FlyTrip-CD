pipeline {
    agent any

    environment {
        // Homebrew paths for macOS / Apple Silicon
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:${env.PATH}"

        // AWS
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'

        // FlyTrip ECR
        ECR_REPOSITORY = 'flytrip-cd'

        // FlyTrip EKS
        EKS_CLUSTER_NAME = 'fly-eks'

        // Docker image
        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        stage('1. Checkout FlyTrip CD Repository') {
            steps {
                checkout scm
            }
        }

        stage('2. Verify Environment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - VERIFYING ENVIRONMENT"
                    echo "=========================================="

                    echo ""
                    echo "Checking AWS CLI..."
                    test -x /opt/homebrew/bin/aws
                    /opt/homebrew/bin/aws --version

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
                    echo "Environment verification passed."
                '''
            }
        }

        stage('3. Verify AWS Connection') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "VERIFYING AWS CONNECTION"
                    echo "=========================================="

                    /opt/homebrew/bin/aws sts get-caller-identity \
                        --region "$AWS_REGION"

                    echo ""
                    echo "AWS connection successful."
                '''
            }
        }

        stage('4. Connect to FlyTrip EKS') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "CONNECTING TO FLYTRIP EKS"
                    echo "=========================================="

                    /opt/homebrew/bin/aws eks update-kubeconfig \
                        --region "$AWS_REGION" \
                        --name "$EKS_CLUSTER_NAME"

                    echo ""
                    echo "Testing Kubernetes connection..."

                    kubectl cluster-info
                    kubectl get nodes

                    echo ""
                    echo "EKS connection successful."
                '''
            }
        }

        stage('5. Deploy Kubernetes Resources') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "DEPLOYING FLYTRIP KUBERNETES RESOURCES"
                    echo "=========================================="

                    echo ""
                    echo "Applying deployment.yaml..."
                    kubectl apply -f deployment.yaml

                    echo ""
                    echo "Applying service.yaml..."
                    kubectl apply -f service.yaml

                    echo ""
                    echo "Kubernetes resources applied successfully."
                '''
            }
        }

        stage('6. Update FlyTrip Application Image') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "UPDATING FLYTRIP APPLICATION IMAGE"
                    echo "=========================================="

                    echo ""
                    echo "ECR Image:"
                    echo "$IMAGE"

                    echo ""
                    echo "Finding FlyTrip deployment..."

                    DEPLOYMENT_NAME=$(kubectl get deployments \
                        -o jsonpath='{.items[0].metadata.name}')

                    if [ -z "$DEPLOYMENT_NAME" ]; then
                        echo "ERROR: No Kubernetes deployment found."
                        exit 1
                    fi

                    echo "Deployment: $DEPLOYMENT_NAME"

                    echo ""
                    echo "Finding container..."

                    CONTAINER_NAME=$(kubectl get deployment "$DEPLOYMENT_NAME" \
                        -o jsonpath='{.spec.template.spec.containers[0].name}')

                    if [ -z "$CONTAINER_NAME" ]; then
                        echo "ERROR: No container found in deployment."
                        exit 1
                    fi

                    echo "Container: $CONTAINER_NAME"

                    echo ""
                    echo "Setting new image..."

                    kubectl set image \
                        deployment/"$DEPLOYMENT_NAME" \
                        "$CONTAINER_NAME"="$IMAGE"

                    echo ""
                    echo "Application image updated successfully."
                '''
            }
        }

        stage('7. Wait for FlyTrip Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "WAITING FOR FLYTRIP DEPLOYMENT"
                    echo "=========================================="

                    DEPLOYMENT_NAME=$(kubectl get deployments \
                        -o jsonpath='{.items[0].metadata.name}')

                    echo "Waiting for deployment:"
                    echo "$DEPLOYMENT_NAME"

                    kubectl rollout status \
                        deployment/"$DEPLOYMENT_NAME" \
                        --timeout=180s

                    echo ""
                    echo "FlyTrip deployment completed successfully."
                '''
            }
        }

        stage('8. Verify FlyTrip Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP DEPLOYMENT VERIFICATION"
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
                    echo "========== DEPLOYMENT STATUS =========="

                    DEPLOYMENT_NAME=$(kubectl get deployments \
                        -o jsonpath='{.items[0].metadata.name}')

                    kubectl get deployment "$DEPLOYMENT_NAME"

                    echo ""
                    echo "FlyTrip Kubernetes verification completed."
                '''
            }
        }
    }

    post {
        success {
            echo '''
==========================================
        FLYTRIP CD PASSED
==========================================

FlyTrip application was successfully
deployed to Amazon EKS.

Cluster: fly-eks
Region:  us-east-1
ECR:     flytrip-cd

==========================================
'''
        }

        failure {
            echo '''
==========================================
        FLYTRIP CD FAILED
==========================================

A previous pipeline stage failed.

The "DEPLOYMENT FAILED" message here is
only the final Jenkins post-action.

Check the FIRST ERROR above this message
in the Jenkins Console Output.

==========================================
'''
        }

        always {
            echo '''
==========================================
        FLYTRIP CD FINISHED
==========================================
'''
        }
    }
}
