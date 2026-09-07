pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'fly-eks'
        ECR_REGISTRY = '584612873567.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'flytrip-cd'
        IMAGE = '584612873567.dkr.ecr.us-east-1.amazonaws.com/flytrip-cd:latest'

        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"
    }

    stages {

        stage('1. Verify FlyTrip Files') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - VERIFYING PROJECT FILES"
                    echo "=========================================="

                    test -f Jenkinsfile
                    test -f deployment.yaml
                    test -f service.yaml

                    echo "Jenkinsfile found."
                    echo "deployment.yaml found."
                    echo "service.yaml found."
                '''
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
                    command -v aws
                    aws --version

                    echo "Checking kubectl..."
                    command -v kubectl
                    kubectl version --client

                    echo "Checking AWS identity..."
                    aws sts get-caller-identity

                    echo "Tools verified successfully."
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

                    aws eks update-kubeconfig \
                        --region "$AWS_REGION" \
                        --name "$EKS_CLUSTER"

                    kubectl config current-context

                    echo "Connected to EKS cluster: $EKS_CLUSTER"
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
                    echo "$IMAGE"

                    kubectl set image deployment/flytrip \
                        flytrip="$IMAGE"

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

                    kubectl rollout status deployment/flytrip \
                        --timeout=180s

                    echo ""
                    echo "FlyTrip deployment completed successfully."
                '''
            }
        }

        stage('7. Verify FlyTrip') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "FLYTRIP - FINAL VERIFICATION"
                    echo "=========================================="

                    echo "Deployment:"
                    kubectl get deployment flytrip

                    echo ""
                    echo "Pods:"
                    kubectl get pods -l app=flytrip

                    echo ""
                    echo "Service:"
                    kubectl get service flytrip-service

                    echo ""
                    echo "=========================================="
                    echo "FLYTRIP DEPLOYMENT SUCCESSFUL"
                    echo "=========================================="
                '''
            }
        }
    }

    post {
        success {
            echo 'FlyTrip CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'FlyTrip CI/CD pipeline failed. Check the stage and console output above.'
        }
    }
}
