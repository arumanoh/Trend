pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('62f99055-c99c-4666-97a5-b7e5acb5e326')
        IMAGE_NAME = 'arumanoh/trend-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
                sh 'docker push $IMAGE_NAME:latest'
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
                sh 'kubectl apply -f k8s/service.yaml'
                sh 'kubectl set image deployment/trend-app trend-app=$IMAGE_NAME:$IMAGE_TAG'
                sh 'kubectl rollout status deployment/trend-app'
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}
