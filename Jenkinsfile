pipeline {
    agent any

    environment {
        // ==============================
        // AWS
        // ==============================
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = '584612873567'

        // ==============================
        // ECR
        // ==============================
        ECR_REPOSITORY  = 'flytrip-cd'

        // ==============================
        // EKS
        // ==============================
        EKS_CLUSTER_NAME = 'fly-eks'

        // ==============================
        // macOS / Homebrew
        // ==============================
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:${env.PATH}"

        AWS_CLI    = '/opt/homebrew/bin/aws'
        KUBECTL    = '/opt/homebrew/bin/kubectl'

        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        // ==========================================
        // 1. CHECKOUT
        // ==========================================
        stage('1. Checkout FlyTrip CD') {
            steps {
                checkout scm
            }
        }

        // ==========================================
        // 2. VERIFY ENVIRONMENT
        // ==========================================
        stage('2. Verify Environment') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "FLYTRIP - VERIFYING ENVIRONMENT"
                    echo "=========================================="

                    echo ""
                    echo "Operating system:"
                    uname -a

                    echo ""
                    echo "AWS CLI:"
                    if [ ! -x "${AWS_CLI}" ]; then
                        echo "ERROR: AWS CLI not found at ${AWS_CLI}"
                        echo "PATH=${PATH}"
                        exit 1
                    fi

                    "${AWS_CLI}" --version

                    echo ""
                    echo "kubectl:"
                    if [ ! -x "${KUBECTL}" ]; then
                        echo "ERROR: kubectl not found at ${KUBECTL}"
                        echo "PATH=${PATH}"
                        echo ""
                        echo "Expected locations:"
                        echo "  /opt/homebrew/bin/kubectl"
                        echo "  /usr/local/bin/kubectl"
                        exit 1
                    fi

                    "${KUBECTL}" version --client

                    echo ""
                    echo "Git:"
                    git --version

                    echo ""
                    echo "Workspace:"
                    pwd
                    ls -la

                    echo ""
                    echo "Kubernetes manifests:"
                    test -f deployment.yaml
                    test -f service.yaml

                    echo "deployment.yaml found"
                    echo "service.yaml found"

                    echo ""
                    echo "=========================================="
                    echo "ENVIRONMENT CHECK PASSED"
                    echo "=========================================="
                '''
            }
        }

        // ==========================================
        // 3. VERIFY AWS CREDENTIALS
        // ==========================================
        stage('3. Verify AWS Access') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "VERIFYING AWS ACCESS"
                    echo "=========================================="

                    "${AWS_CLI}" sts get-caller-identity

                    echo ""
                    echo "AWS access is working."
                '''
            }
        }

        // ==========================================
        // 4. CONNECT TO EKS
        // ==========================================
        stage('4. Connect to FlyTrip EKS') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "CONNECTING TO EKS"
                    echo "=========================================="

                    echo "Cluster: ${EKS_CLUSTER_NAME}"
                    echo "Region:  ${AWS_REGION}"

                    "${AWS_CLI}" eks update-kubeconfig \
                        --region "${AWS_REGION}" \
                        --name "${EKS_CLUSTER_NAME}"

                    echo ""
                    echo "Testing Kubernetes connection..."

                    "${KUBECTL}" cluster-info

                    echo ""
                    echo "Kubernetes nodes:"
                    "${KUBECTL}" get nodes -o wide

                    echo ""
                    echo "=========================================="
                    echo "EKS CONNECTION PASSED"
                    echo "=========================================="
                '''
            }
        }

        // ==========================================
        // 5. VALIDATE MANIFESTS
        // ==========================================
        stage('5. Validate Kubernetes Manifests') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "VALIDATING KUBERNETES MANIFESTS"
                    echo "=========================================="

                    echo ""
                    echo "Checking deployment.yaml..."

                    "${KUBECTL}" apply \
                        --dry-run=client \
                        -f deployment.yaml

                    echo ""
                    echo "Checking service.yaml..."

                    "${KUBECTL}" apply \
                        --dry-run=client \
                        -f service.yaml

                    echo ""
                    echo "Checking for obsolete restaurant-company references..."

                    if grep -Rni \
                        --exclude-dir=.git \
                        "restaurant-company" \
                        deployment.yaml \
                        service.yaml \
                        Jenkinsfile 2>/dev/null; then

                        echo ""
                        echo "ERROR: Old restaurant-company reference detected."
                        echo "Please remove old application references."
                        exit 1
                    fi

                    echo ""
                    echo "Manifest validation passed."
                '''
            }
        }

        // ==========================================
        // 6. DEPLOY KUBERNETES RESOURCES
        // ==========================================
        stage('6. Deploy FlyTrip Kubernetes Resources') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "DEPLOYING FLYTRIP TO KUBERNETES"
                    echo "=========================================="

                    echo ""
                    echo "Applying deployment.yaml..."
                    "${KUBECTL}" apply -f deployment.yaml

                    echo ""
                    echo "Applying service.yaml..."
                    "${KUBECTL}" apply -f service.yaml

                    echo ""
                    echo "Kubernetes resources applied."
                '''
            }
        }

        // ==========================================
        // 7. UPDATE APPLICATION IMAGE
        // ==========================================
        stage('7. Update FlyTrip Application Image') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "UPDATING APPLICATION IMAGE"
                    echo "=========================================="

                    echo ""
                    echo "Target image:"
                    echo "${IMAGE}"

                    echo ""
                    echo "Current deployments:"
                    "${KUBECTL}" get deployments

                    echo ""
                    echo "Updating deployment image..."

                    "${KUBECTL}" set image \
                        deployment/flytrip \
                        flytrip="${IMAGE}"

                    echo ""
                    echo "Image update completed."
                '''
            }
        }

        // ==========================================
        // 8. WAIT FOR ROLLOUT
        // ==========================================
        stage('8. Wait for FlyTrip Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "WAITING FOR DEPLOYMENT"
                    echo "=========================================="

                    "${KUBECTL}" rollout status \
                        deployment/flytrip \
                        --timeout=180s

                    echo ""
                    echo "Rollout completed successfully."
                '''
            }
        }

        // ==========================================
        // 9. VERIFY
        // ==========================================
        stage('9. Verify FlyTrip Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "=========================================="
                    echo "VERIFYING FLYTRIP"
                    echo "=========================================="

                    echo ""
                    echo "========== DEPLOYMENTS =========="
                    "${KUBECTL}" get deployments -o wide

                    echo ""
                    echo "========== PODS =========="
                    "${KUBECTL}" get pods -o wide

                    echo ""
                    echo "========== SERVICES =========="
                    "${KUBECTL}" get services

                    echo ""
                    echo "========== DEPLOYMENT STATUS =========="
                    "${KUBECTL}" describe deployment flytrip --tail=50

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
FLYTRIP CD SUCCESS
==========================================
Application deployed successfully.

EKS Cluster: fly-eks
AWS Region:  us-east-1
ECR Image:   ${IMAGE}
==========================================
'''
        }

        failure {
            echo '''
==========================================
FLYTRIP CD FAILED
==========================================
The FAILED message is only the result.

Look ABOVE in the Jenkins console for
the FIRST command that returned exit code 1.
==========================================
'''
        }

        always {
            echo 'FlyTrip CD pipeline finished.'
        }
    }
}
