pipeline {
    agent any

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'a1e43dd78f84f53e38c98c0c9fd2e26b37b06598',
            description: 'Docker image tag from FlyTrip CI / ECR'
        )
    }

    environment {
        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '584612873567'
        ECR_REPOSITORY = 'flytrip-ci'

        EKS_CLUSTER_NAME = 'fly-eks'

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}"

        DEPLOYMENT_NAME = 'flytrip'
        CONTAINER_NAME = 'flytrip'
        SERVICE_NAME = 'flytrip-service'
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
                    echo "=========================================="
                    echo "Checking required tools"
                    echo "=========================================="

                    echo "AWS CLI:"
                    aws --version

                    echo ""
                    echo "kubectl:"
                    kubectl version --client

                    echo ""
                    echo "Repository files:"
                    ls -la

                    echo ""
                    echo "Deployment files:"
                    ls -l deployment.yaml service.yaml
                '''
            }
        }

        stage('3. Verify AWS Connection') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "AWS Authentication"
                    echo "=========================================="

                    aws sts get-caller-identity \
                        --region ${AWS_REGION}
                '''
            }
        }

        stage('4. Verify Image in ECR') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Checking Docker image in ECR"
                    echo "=========================================="

                    echo "Repository:"
                    echo "${ECR_REPOSITORY}"

                    echo "Image tag:"
                    echo "${IMAGE_TAG}"

                    echo "Full image:"
                    echo "${IMAGE}"

                    aws ecr describe-images \
                        --repository-name ${ECR_REPOSITORY} \
                        --image-ids imageTag=${IMAGE_TAG} \
                        --region ${AWS_REGION}

                    echo ""
                    echo "Image exists in ECR."
                '''
            }
        }

        stage('5. Connect to EKS') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Connecting to EKS"
                    echo "=========================================="

                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER_NAME}

                    echo ""
                    echo "Testing Kubernetes connection..."

                    kubectl get nodes
                '''
            }
        }

        stage('6. Deploy Kubernetes Resources') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Applying Kubernetes resources"
                    echo "=========================================="

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml

                    echo ""
                    echo "Kubernetes resources applied."
                '''
            }
        }

        stage('7. Update Application Image') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Updating FlyTrip image"
                    echo "=========================================="

                    echo "Deployment: ${DEPLOYMENT_NAME}"
                    echo "Container: ${CONTAINER_NAME}"
                    echo "Image: ${IMAGE}"

                    kubectl set image \
                        deployment/${DEPLOYMENT_NAME} \
                        ${CONTAINER_NAME}=${IMAGE}

                    echo ""
                    echo "Image update completed."
                '''
            }
        }

        stage('8. Wait for Deployment') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    sh '''
                        echo "=========================================="
                        echo "Waiting for rollout"
                        echo "=========================================="

                        kubectl rollout status \
                            deployment/${DEPLOYMENT_NAME} \
                            --timeout=180s

                        echo ""
                        echo "Rollout completed successfully."
                    '''
                }
            }
        }

        stage('9. Verify Deployment') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "DEPLOYMENT VERIFICATION"
                    echo "=========================================="

                    echo ""
                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment ${DEPLOYMENT_NAME}

                    echo ""
                    echo "========== SERVICE =========="
                    kubectl get service ${SERVICE_NAME}

                    echo ""
                    echo "========== IMAGE =========="
                    kubectl get deployment ${DEPLOYMENT_NAME} \
                        -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo ""
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

FlyTrip has been successfully deployed
to Amazon EKS.

EKS Cluster: fly-eks
ECR Repository: flytrip-ci
Image: ${IMAGE}
==========================================
'''
        }

        failure {
            echo '''
==========================================
FLYTRIP CD FAILED
==========================================

Check:
1. Jenkins AWS credentials
2. ECR image/tag
3. EKS cluster
4. deployment.yaml
5. service.yaml
6. Kubernetes events
==========================================
'''
        }
    }
}
