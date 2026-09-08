pipeline {
    agent any

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:${env.PATH}"

        AWS_REGION = "us-east-1"
        EKS_CLUSTER = "fly-eks"

        ECR_REGISTRY = "584612873567.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPOSITORY = "flytrip-ci"

        DEPLOYMENT_NAME = "flytrip"
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo 'Checking out FlyTrip CD repository...'
                checkout scm
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    echo "========== VERIFY TOOLS =========="

                    echo "AWS CLI:"
                    aws --version

                    echo "kubectl:"
                    kubectl version --client

                    echo "Git:"
                    git --version
                '''
            }
        }

        stage('3. AWS Authentication') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "========== AWS AUTHENTICATION =========="
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('4. Connect to EKS') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "========== CONNECT TO EKS =========="

                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $EKS_CLUSTER

                        echo "Current context:"
                        kubectl config current-context

                        echo "Cluster nodes:"
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('5. Find Latest CI Image') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        env.IMAGE_TAG = sh(
                            script: '''
                                aws ecr describe-images \
                                    --repository-name "$ECR_REPOSITORY" \
                                    --region "$AWS_REGION" \
                                    --query 'sort_by(imageDetails,& imagePushedAt)[-1].imageTags[0]' \
                                    --output text
                            ''',
                            returnStdout: true
                        ).trim()

                        if (!env.IMAGE_TAG || env.IMAGE_TAG == 'None') {
                            error("No image tag found in ECR repository ${env.ECR_REPOSITORY}")
                        }

                        env.FULL_IMAGE = "${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}:${env.IMAGE_TAG}"

                        echo "Deploying image:"
                        echo "${env.FULL_IMAGE}"
                    }
                }
            }
        }

        stage('6. Deploy Kubernetes Resources') {
            steps {
                sh '''
                    echo "========== DEPLOY =========="

                    if [ -f namespace.yaml ]; then
                        kubectl apply -f namespace.yaml
                    fi

                    if [ -f configmap.yaml ]; then
                        kubectl apply -f configmap.yaml
                    fi

                    if [ -f secrets.yaml ]; then
                        kubectl apply -f secrets.yaml
                    fi

                    if [ -f serviceaccount.yaml ]; then
                        kubectl apply -f serviceaccount.yaml
                    fi

                    kubectl apply -f deployment.yaml

                    if [ -f service.yaml ]; then
                        kubectl apply -f service.yaml
                    fi

                    if [ -f networkpolicy.yaml ]; then
                        kubectl apply -f networkpolicy.yaml
                    fi

                    if [ -f pdb.yaml ]; then
                        kubectl apply -f pdb.yaml
                    fi

                    if [ -f hpa.yaml ]; then
                        kubectl apply -f hpa.yaml
                    fi

                    if [ -f ingress.yaml ]; then
                        kubectl apply -f ingress.yaml
                    fi
                '''
            }
        }

        stage('7. Deploy CI Image') {
            steps {
                sh '''
                    echo "========== UPDATE IMAGE =========="
                    echo "Using: $FULL_IMAGE"

                    kubectl set image \
                        deployment/$DEPLOYMENT_NAME \
                        flytrip=$FULL_IMAGE
                '''
            }
        }

        stage('8. Verify Rollout') {
            steps {
                sh '''
                    echo "========== VERIFY ROLLOUT =========="

                    kubectl rollout status \
                        deployment/$DEPLOYMENT_NAME \
                        --timeout=300s
                '''
            }
        }

        stage('9. Verify Deployment') {
            steps {
                sh '''
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment $DEPLOYMENT_NAME

                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo "========== SERVICES =========="
                    kubectl get services

                    echo "========== DEPLOYED IMAGE =========="
                    kubectl get deployment $DEPLOYMENT_NAME \
                        -o jsonpath='{.spec.template.spec.containers[0].image}{"\\n"}'
                '''
            }
        }
    }

    post {

        success {
            echo '=========================================='
            echo 'FLYTRIP DEPLOYMENT SUCCESSFUL'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo 'FLYTRIP DEPLOYMENT FAILED'
            echo '=========================================='

            sh '''
                echo "========== PODS =========="
                kubectl get pods || true

                echo "========== EVENTS =========="
                kubectl get events \
                    --sort-by=.metadata.creationTimestamp \
                    | tail -30 || true
            '''
        }

        always {
            echo 'Pipeline finished.'
        }
    }
}
