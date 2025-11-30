pipeline {
    agent any

    environment {
        APP_NAME   = 'mavenapp'
        IMAGE_NAME = 'mavenapp'
        CONTAINER_PORT = '8081'
        HOST_PORT      = '8082'   // Jenkins app port
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy Docker Container') {
            steps {
                sh """
                docker rm -f ${APP_NAME} || true

                docker run -d --name ${APP_NAME} \
                  -p ${HOST_PORT}:${CONTAINER_PORT} \
                  ${IMAGE_NAME}:${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        success {
            echo "App deployed successfully at: http://localhost:${HOST_PORT}"
        }
    }
}
