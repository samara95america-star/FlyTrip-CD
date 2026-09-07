pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-ci'
        EKS_CLUSTER_NAME = 'fly-eks'

        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:${env.PATH}"

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        stage('1. Checkout CD Repository') {
            steps {
                checkout scm
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    echo "========== AWS CLI =========="
                    aws --version

                    echo "========== kubectl =========="
                    kubectl version --client

                    echo "========== Files =========="
                    ls -la

                    echo "========== IMAGE =========="
                    echo ${IMAGE}
                '''
            }
        }

        stage('3. Verify AWS Connection') {
            steps {
                sh '''
                    echo "========== AWS ACCOUNT =========="
                    aws sts get-caller-identity

                    echo "========== AWS REGION =========="
                    aws configure get region || true
                '''
            }
        }

        stage('4. Verify Image in ECR') {
            steps {
                sh '''
                    echo "========== ECR IMAGE =========="

                    aws ecr describe-images \
                      --repository-name ${ECR_REPOSITORY} \
                      --region ${AWS_REGION}

                    echo "ECR repository:"
                    echo ${ECR_REPOSITORY}
                '''
            }
        }

        stage('5. Connect to EKS') {
            steps {
                sh '''
                    echo "========== CONNECTING TO EKS =========="

                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER_NAME}

                    echo "========== KUBERNETES NODES =========="

                    kubectl get nodes
                '''
            }
        }

        stage('6. Deploy Kubernetes Resources') {
            steps {
                sh '''
                    echo "========== DEPLOYING KUBERNETES =========="

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml
                '''
            }
        }

        stage('7. Update Application Image') {
            steps {
                sh '''
                    echo "========== DEPLOYING IMAGE =========="
                    echo ${IMAGE}

                    kubectl set image \
                      deployment/flytrip \
                      flytrip=${IMAGE}

                    echo "Image updated successfully."
                '''
            }
        }

        stage('8. Wait for Deployment') {
            steps {
                sh '''
                    echo "========== WAITING FOR DEPLOYMENT =========="

                    kubectl rollout status \
                      deployment/flytrip \
                      --timeout=180s
                '''
            }
        }

        stage('9. Verify Deployment') {
            steps {
                sh '''
                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment flytrip

                    echo "========== SERVICE =========="
                    kubectl get service flytrip-service

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
==========================================
FLYTRIP CD PASSED
Application deployed successfully to EKS
==========================================
'''
        }

        failure {
            echo '''
==========================================
FLYTRIP CD FAILED
Check the Jenkins stage that failed.
==========================================
'''
        }
    }
}
