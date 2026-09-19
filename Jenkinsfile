pipeline {

    agent any

    environment {

        AWS_REGION = 'ap-south-1'

        AWS_ACCOUNT_ID = sh(
            script: "aws sts get-caller-identity --query Account --output text",
            returnStdout: true
        ).trim()

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com"

        BUILD_TAG_VALUE = "${BUILD_NUMBER}"

    }

    stages {

        stage('Checkout') {

            steps {

                checkout scm

            }
        }

        stage('Build User Service') {

            steps {

                dir('user-service') {

                    sh 'mvn clean package -DskipTests'

                }
            }
        }

        stage('Build Order Service') {

            steps {

                dir('order-service') {

                    sh 'mvn clean package -DskipTests'

                }
            }
        }

        stage('Docker Build') {

            steps {

                sh '''
                    docker build \
                    -t $ECR_REGISTRY/user-service:$BUILD_TAG_VALUE \
                    user-service

                    docker build \
                    -t $ECR_REGISTRY/order-service:$BUILD_TAG_VALUE \
                    order-service
                '''
            }
        }

        stage('Login to ECR') {

            steps {

                sh '''
                    aws ecr get-login-password \
                    --region $AWS_REGION \
                    | docker login \
                    --username AWS \
                    --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Push Images') {

            steps {

                sh '''
                    docker push \
                    $ECR_REGISTRY/user-service:$BUILD_TAG_VALUE

                    docker push \
                    $ECR_REGISTRY/order-service:$BUILD_TAG_VALUE
                '''
            }
        }

        stage('Deploy to EKS') {

            steps {

                sh '''
                    aws eks update-kubeconfig \
                    --region $AWS_REGION \
                    --name project15-eks

                    kubectl apply \
                    -f k8s/namespace.yaml

                    sed "s|YOUR_ACCOUNT_ID|$AWS_ACCOUNT_ID|g; s|:1.0|:$BUILD_TAG_VALUE|g" \
                    k8s/user-deployment.yaml \
                    > /tmp/user-deployment.yaml

                    sed "s|YOUR_ACCOUNT_ID|$AWS_ACCOUNT_ID|g; s|:1.0|:$BUILD_TAG_VALUE|g" \
                    k8s/order-deployment.yaml \
                    > /tmp/order-deployment.yaml

                    kubectl apply \
                    -f /tmp/user-deployment.yaml

                    kubectl apply \
                    -f k8s/user-service.yaml

                    kubectl apply \
                    -f /tmp/order-deployment.yaml

                    kubectl apply \
                    -f k8s/order-service.yaml
                '''
            }
        }

        stage('Verify Deployment') {

            steps {

                sh '''
                    kubectl rollout status \
                    deployment/user-service \
                    -n microservices

                    kubectl rollout status \
                    deployment/order-service \
                    -n microservices

                    kubectl get pods \
                    -n microservices

                    kubectl get services \
                    -n microservices
                '''
            }
        }
    }

    post {

        success {

            echo 'Deployment completed successfully!'

        }

        failure {

            echo 'Pipeline failed. Check the stage logs.'

        }

    }
}
