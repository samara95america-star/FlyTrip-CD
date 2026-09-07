pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'

        ECR_REPOSITORY = 'flytrip-cd'
        ECR_REGISTRY   = '584612873567.dkr.ecr.us-east-1.amazonaws.com'
        IMAGE_NAME     = '584612873567.dkr.ecr.us-east-1.amazonaws.com/flytrip-cd:latest'

        EKS_CLUSTER    = 'fly-eks'
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo '========================================'
                echo 'FLYTRIP - CHECKOUT'
                echo '========================================'

                checkout scm
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - VERIFYING TOOLS"
                    echo "========================================"

                    # Jenkins on macOS may not have Homebrew paths by default
                    export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:$PATH"

                    echo ""
                    echo "Checking AWS CLI..."
                    if ! command -v aws >/dev/null 2>&1; then
                        echo "ERROR: AWS CLI was not found."
                        echo "Expected locations:"
                        echo "  /opt/homebrew/bin/aws"
                        echo "  /usr/local/bin/aws"
                        exit 1
                    fi
                    aws --version

                    echo ""
                    echo "Checking kubectl..."
                    if ! command -v kubectl >/dev/null 2>&1; then
                        echo "ERROR: kubectl was not found."
                        exit 1
                    fi
                    kubectl version --client

                    echo ""
                    echo "Checking AWS credentials..."
                    aws sts get-caller-identity

                    echo ""
                    echo "All required tools are available."
                '''
            }
        }

        stage('3. Connect to EKS') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - CONNECTING TO EKS"
                    echo "========================================"

                    export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:$PATH"

                    aws eks update-kubeconfig \
                        --region "$AWS_REGION" \
                        --name "$EKS_CLUSTER"

                    echo ""
                    echo "Connected to EKS cluster:"
                    echo "$EKS_CLUSTER"

                    echo ""
                    kubectl cluster-info
                '''
            }
        }

        stage('4. Validate Kubernetes Files') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - VALIDATING KUBERNETES FILES"
                    echo "========================================"

                    test -f deployment.yaml
                    test -f service.yaml

                    echo "deployment.yaml found."
                    echo "service.yaml found."

                    echo ""
                    echo "Checking for old restaurant-company references..."

                    if grep -R "restaurant-company" deployment.yaml service.yaml Jenkinsfile; then
                        echo ""
                        echo "ERROR: restaurant-company reference still exists."
                        echo "Please remove it before deployment."
                        exit 1
                    fi

                    echo ""
                    echo "No restaurant-company references found."

                    echo ""
                    echo "Kubernetes manifests:"
                    kubectl apply --dry-run=client -f deployment.yaml
                    kubectl apply --dry-run=client -f service.yaml

                    echo ""
                    echo "Kubernetes files are valid."
                '''
            }
        }

        stage('5. Deploy FlyTrip Kubernetes Resources') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - DEPLOYING KUBERNETES RESOURCES"
                    echo "========================================"

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml

                    echo ""
                    echo "Kubernetes resources applied successfully."
                '''
            }
        }

        stage('6. Update FlyTrip Application Image') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - UPDATING APPLICATION IMAGE"
                    echo "========================================"

                    echo "Deployment: flytrip"
                    echo "Container: flytrip"
                    echo "Image:"
                    echo "$IMAGE_NAME"

                    kubectl set image deployment/flytrip \
                        flytrip="$IMAGE_NAME"

                    echo ""
                    echo "Application image updated successfully."
                '''
            }
        }

        stage('7. Wait for FlyTrip Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - WAITING FOR DEPLOYMENT"
                    echo "========================================"

                    kubectl rollout status deployment/flytrip \
                        --timeout=180s

                    echo ""
                    echo "FlyTrip deployment completed successfully."
                '''
            }
        }

        stage('8. Final Verification') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "FLYTRIP - FINAL VERIFICATION"
                    echo "========================================"

                    echo ""
                    echo "Deployment:"
                    kubectl get deployment flytrip

                    echo ""
                    echo "Pods:"
                    kubectl get pods -l app=flytrip

                    echo ""
                    echo "Service:"
                    kubectl get service flytrip-service

                    echo ""
                    echo "========================================"
                    echo "FLYTRIP DEPLOYMENT SUCCESSFUL!"
                    echo "========================================"
                '''
            }
        }
    }

    post {
        success {
            echo 'FLYTRIP CI/CD PIPELINE FINISHED SUCCESSFULLY.'
        }

        failure {
            echo 'FLYTRIP CI/CD PIPELINE FAILED.'
            echo 'Check the failed stage above for the exact error.'
        }
    }
}
