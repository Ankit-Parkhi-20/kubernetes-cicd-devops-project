pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-1'
        ECR_REPO = 'devops-k8s-demo'
        EKS_CLUSTER = 'devops-k8s-demo'
        K8S_NAMESPACE = 'devops-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Test') {
            steps {
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install -r app/requirements.txt'
                sh '.venv/bin/python -m pytest -q'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${ECR_REPO}:${IMAGE_TAG} .'
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set -e

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                        ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker tag ${ECR_REPO}:${IMAGE_TAG} \
                        ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}

                        docker push \
                        ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Prepare Kubernetes Manifest') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set -e

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                        ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        IMAGE_URI=${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}

                        sed "s|IMAGE_PLACEHOLDER|${IMAGE_URI}|g" \
                        k8s/deployment.yaml > k8s/deployment-rendered.yaml

                        echo "Kubernetes image:"
                        grep "image:" k8s/deployment-rendered.yaml
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set -e

                        mkdir -p "$WORKSPACE/.kube"

                        export KUBECONFIG="$WORKSPACE/.kube/config"

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER} \
                          --kubeconfig "$KUBECONFIG"

                        kubectl apply -f k8s/namespace.yaml

                        kubectl apply -f k8s/deployment-rendered.yaml

                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status \
                          deployment/${ECR_REPO} \
                          -n ${K8S_NAMESPACE} \
                          --timeout=180s

                        echo "Pods:"
                        kubectl get pods -n ${K8S_NAMESPACE}

                        echo "Service:"
                        kubectl get svc -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the console logs.'
        }
    }
}