pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-cd'
        EKS_CLUSTER_NAME = 'fly-eks'

        // macOS Homebrew paths
        PATH+TOOLS = '/opt/homebrew/bin:/usr/local/bin'

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
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - VERIFYING ENVIRONMENT"
                    echo "=========================================="

                    echo ""
                    echo "PATH:"
                    echo "$PATH"

                    echo ""
                    echo "Checking AWS CLI..."

                    if command -v aws >/dev/null 2>&1; then
                        echo "AWS CLI found:"
                        command -v aws
                        aws --version
                    else
                        echo "ERROR: AWS CLI was not found."
                        echo "Expected locations:"
                        echo "  /opt/homebrew/bin/aws"
                        echo "  /usr/local/bin/aws"
                        exit 1
                    fi

                    echo ""
                    echo "Checking kubectl..."

                    if command -v kubectl >/dev/null 2>&1; then
                        echo "kubectl found:"
                        command -v kubectl
                        kubectl version --client
                    else
                        echo "ERROR: kubectl was not found."
                        echo "Expected locations:"
                        echo "  /usr/local/bin/kubectl"
                        echo "  /opt/homebrew/bin/kubectl"
                        exit 1
                    fi

                    echo ""
                    echo "Checking AWS credentials..."

                    aws sts get-caller-identity

                    echo ""
                    echo "Checking project files..."

                    test -f deployment.yaml
                    test -f service.yaml

                    echo "deployment.yaml found"
                    echo "service.yaml found"

                    echo ""
                    echo "Checking for old restaurant-company references..."

                    if grep -R "restaurant-company" deployment.yaml service.yaml Jenkinsfile; then
                        echo "ERROR: Old restaurant-company reference found!"
                        exit 1
                    fi

                    echo "No restaurant-company references found."

                    echo ""
                    echo "Environment verification completed successfully."
                '''
            }
        }

        stage('3. Connect to EKS') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - CONNECTING TO EKS"
                    echo "=========================================="

                    aws eks update-kubeconfig \
                        --region "${AWS_REGION}" \
                        --name "${EKS_CLUSTER_NAME}"

                    echo ""
                    echo "Testing Kubernetes connection..."

                    kubectl cluster-info

                    echo ""
                    echo "Kubernetes nodes:"

                    kubectl get nodes
                '''
            }
        }

        stage('4. Validate Kubernetes Files') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - VALIDATING KUBERNETES FILES"
                    echo "=========================================="

                    kubectl apply \
                        --dry-run=client \
                        -f deployment.yaml

                    kubectl apply \
                        --dry-run=client \
                        -f service.yaml

                    echo ""
                    echo "Kubernetes YAML validation successful."
                '''
            }
        }

        stage('5. Deploy FlyTrip Kubernetes Resources') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - DEPLOYING KUBERNETES RESOURCES"
                    echo "=========================================="

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml

                    echo ""
                    echo "Current FlyTrip resources:"

                    kubectl get deployment flytrip
                    kubectl get service flytrip-service

                    echo ""
                    echo "Kubernetes resources deployed successfully."
                '''
            }
        }

        stage('6. Update FlyTrip Application Image') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - UPDATING APPLICATION IMAGE"
                    echo "=========================================="

                    echo "Image:"
                    echo "${IMAGE}"

                    kubectl set image \
                        deployment/flytrip \
                        flytrip="${IMAGE}"

                    echo ""
                    echo "Application image updated successfully."
                '''
            }
        }

        stage('7. Wait for FlyTrip Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - WAITING FOR ROLLOUT"
                    echo "=========================================="

                    kubectl rollout status \
                        deployment/flytrip \
                        --timeout=180s

                    echo ""
                    echo "FlyTrip rollout completed successfully."
                '''
            }
        }

        stage('8. Verify FlyTrip Deployment') {
            steps {
                sh '''
                    set -eu

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
                    echo "========== IMAGE =========="
                    kubectl get deployment flytrip \
                        -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo ""
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
==========================================
Application deployed successfully to EKS.
'''
        }

        failure {
            echo '''
==========================================
FLYTRIP CD FAILED
==========================================
Check the failed Jenkins stage above.
'''
        }
    }
}
