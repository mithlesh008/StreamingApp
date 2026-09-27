pipeline {
  agent any
  environment { AWS_REGION = 'ap-south-1'; IMAGE_TAG = "1.0.${BUILD_NUMBER}" }
  triggers { githubPush() }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build and Push ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
          sh '''
            set -euxo pipefail
            ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
            REGISTRY="$ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com"
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$REGISTRY"
            docker build -t "$REGISTRY/streaming-auth:$IMAGE_TAG" backend/authService
            docker build -t "$REGISTRY/streaming-stream:$IMAGE_TAG" -f backend/streamingService/Dockerfile backend
            docker build -t "$REGISTRY/streaming-admin:$IMAGE_TAG" -f backend/adminService/Dockerfile backend
            docker build -t "$REGISTRY/streaming-chat:$IMAGE_TAG" -f backend/chatService/Dockerfile backend
            docker build -t "$REGISTRY/streaming-frontend:$IMAGE_TAG" frontend
            for repo in streaming-auth streaming-stream streaming-admin streaming-chat streaming-frontend; do docker push "$REGISTRY/$repo:$IMAGE_TAG"; done
          '''
        }
      }
    }
  }
  post { success { echo 'All five images pushed to ECR.' } }
}
