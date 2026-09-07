pipeline {
    agent any

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:${env.PATH}"

        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'

        ECR_REPOSITORY = 'flytrip-ci'
        ECR_URI = '584612873567.dkr.ecr.us-east-1.amazonaws.com/flytrip-ci'

        EKS_CLUSTER_NAME = 'fly-eks'
    }

    stages {

        stage('1. Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Check AWS CLI') {
            steps {
                sh '''
                    set -e

                    echo "AWS location:"
                    which aws

                    echo "AWS version:"
                    aws --version
                '''
            }
        }

        stage('3. Check AWS Connection') {
            steps {
                sh '''
                    set -e

                    echo "AWS account:"
                    aws sts get-caller-identity

                    echo "Region:"
                    echo ${AWS_REGION}
                '''
            }
        }

        stage('4. Check ECR') {
            steps {
                sh '''
                    set -e

                    echo "Checking ECR repository..."

                    aws ecr describe-repositories \
                        --repository-names ${ECR_REPOSITORY} \
                        --region ${AWS_REGION}

                    echo "Checking image..."

                    aws ecr describe-images \
                        --repository-name ${ECR_REPOSITORY} \
                        --region ${AWS_REGION} \
                        --query 'imageDetails[*].imageTags' \
                        --output table
                '''
            }
        }

        stage('5. Connect to EKS') {
            steps {
                sh '''
                    set -e

                    echo "Connecting to EKS..."

                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER_NAME}

                    echo "Testing Kubernetes..."

                    kubectl get nodes
                '''
            }
        }

        stage('6. Deploy FlyTrip') {
            steps {
                sh '''
                    set -e

                    echo "Applying Kubernetes files..."

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml
                '''
            }
        }

        stage('7. Wait for Deployment') {
            steps {
                sh '''
                    set -e

                    kubectl rollout status \
                        deployment/flytrip \
                        --timeout=180s
                '''
            }
        }

        stage('8. Verify') {
            steps {
                sh '''
                    set -e

                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment flytrip

                    echo "========== SERVICE =========="
                    kubectl get service flytrip

                    echo "========== IMAGE =========="
                    kubectl get deployment flytrip \
                        -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo
                '''
            }
        }
    }

    post {
        success {
            echo '''
==================================================
              FLYTRIP CD SUCCESS
==================================================
Jenkins Build Now completed successfully.

ECR:
584612873567.dkr.ecr.us-east-1.amazonaws.com/flytrip-ci

EKS:
fly-eks
==================================================
'''
        }

        failure {
            echo '''
==================================================
              FLYTRIP CD FAILED
==================================================
Look at the FIRST stage that failed.
==================================================
'''
        }
    }
}
