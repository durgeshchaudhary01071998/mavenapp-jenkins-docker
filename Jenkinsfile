pipeline {
    agent any

    environment {
        APP_NAME = 'mavenapp-deployment'
        IMAGE    = 'mavenapp'
        PORT     = '8081'
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Docker Image Build') {
            steps {
                sh """
                docker build -t ${IMAGE}:${BUILD_NUMBER} .
                docker tag ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest
                """
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                sh """
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
                kubectl rollout status deployment/mavenapp-deployment
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful → App URL: http://localhost:30080"
        }
        failure {
            echo "❌ Deployment/Build failed – check logs"
        }
    }
}
