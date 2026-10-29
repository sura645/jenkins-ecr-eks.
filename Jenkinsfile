
pipeline {
    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        ECR_REPOSITORY = 'jenkins-demo'
        EKS_CLUSTER    = 'jenkins-demo'
        APP_NAME       = 'jenkins-demo'
        CONTAINER_NAME = 'web'
        KUBECONFIG     = '/var/lib/jenkins/.kube/config'
    }

    stages {

        stage('Check AWS Access') {
            steps {
                sh '''
                    set -e
                    aws sts get-caller-identity
                '''
                script {
                    env.AWS_ACCOUNT_ID = sh(
                        script: 'aws sts get-caller-identity --query Account --output text',
                        returnStdout: true
                    ).trim()

                    env.ECR_URI = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPOSITORY}"

                    echo "AWS Account: ${env.AWS_ACCOUNT_ID}"
                    echo "ECR Repository: ${env.ECR_URI}"
                }
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
                sh '''
                    set -e
                    echo "Workspace: $WORKSPACE"
                    ls -la
                    test -f Dockerfile
                    test -f k8s/deployment.yaml
                    test -f k8s/service.yaml
                '''
            }
        }

        stage('Verify ECR Repository') {
            steps {
                sh '''
                    set -e
                    aws ecr describe-repositories \
                      --repository-names "$ECR_REPOSITORY" \
                      --region "$AWS_REGION"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e
                    docker build \
                      -t "$ECR_URI:$BUILD_NUMBER" \
                      -t "$ECR_URI:latest" \
                      .
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    set -e
                    aws ecr get-login-password --region "$AWS_REGION" |
                    docker login \
                      --username AWS \
                      --password-stdin \
                      "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    set -e
                    docker push "$ECR_URI:$BUILD_NUMBER"
                    docker push "$ECR_URI:latest"
                '''
            }
        }

        stage('Connect to EKS') {
            steps {
                sh '''
                    set -e
                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER"

                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -e

                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl set image \
                      "deployment/$APP_NAME" \
                      "$CONTAINER_NAME=$ECR_URI:$BUILD_NUMBER"

                    kubectl rollout status \
                      "deployment/$APP_NAME" \
                      --timeout=180s

                    kubectl get deployment "$APP_NAME"
                    kubectl get pods -o wide
                    kubectl get services
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Docker image pushed to ECR and deployment completed.'
        }

        failure {
            echo 'FAILED: Check the Jenkins Console Output for the failing stage.'
        }

        always {
            sh '''
                docker logout "${AWS_ACCOUNT_ID:-invalid}.dkr.ecr.${AWS_REGION}.amazonaws.com" || true
            '''
        }
    }
}
