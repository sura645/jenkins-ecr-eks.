pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPOSITORY = 'jenkins-demo'
        EKS_CLUSTER = 'jenkins-demo'
        APP_NAME = 'jenkins-demo'
        CONTAINER_NAME = 'web'
    }

    stages {
        stage('Check AWS Access') {
            steps {
                script {
                    env.AWS_ACCOUNT_ID = sh(
                        script: 'aws sts get-caller-identity --query Account --output text',
                        returnStdout: true
                    ).trim()

                    env.ECR_URI = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPOSITORY}"
                }
                sh 'aws sts get-caller-identity'
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${ECR_URI}:${BUILD_NUMBER} .'
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} |
                    docker login --username AWS --password-stdin \
                    ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh 'docker push ${ECR_URI}:${BUILD_NUMBER}'
            }
        }

        stage('Connect to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER}

                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl set image \
                      deployment/${APP_NAME} \
                      ${CONTAINER_NAME}=${ECR_URI}:${BUILD_NUMBER}

                    kubectl rollout status \
                      deployment/${APP_NAME} \
                      --timeout=180s
                '''
            }
        }
    }
}
