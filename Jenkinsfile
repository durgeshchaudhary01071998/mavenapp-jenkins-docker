pipeline {
    agent any

    environment {
        // App / image
        APP_NAME        = 'mavenapp'
        IMAGE_NAME      = 'mavenapp'

        // Kubernetes
        K8S_NAMESPACE   = 'default'
        K8S_DEPLOYMENT  = 'mavenapp-deployment'
        KUBECONFIG_PATH = '/var/jenkins_home/.kube/config'

        // App ports
        CONTAINER_PORT  = '8081'   // Spring Boot app inside container
        SERVICE_PORT    = '30080'  // NodePort (K8s service)

        // Flags to enable/disable optional stages
        SONARQUBE_ENABLED   = 'false'     // set to 'true' when SonarQube is ready
        SONARQUBE_SERVER    = 'Local-Sonar' // Jenkins Sonar server name

        OWASP_ENABLED       = 'false'     // set to 'true' when OWASP plugin is ready

        TRIVY_FAIL_ON_HIGH  = 'false'     // set 'true' to fail pipeline on HIGH/CRITICAL
    }

    options {
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build & Unit Tests') {
            steps {
                sh 'mvn -B clean test'
                sh 'mvn -B package -DskipTests'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { env.SONARQUBE_ENABLED == 'true' }
            }
            steps {
                withSonarQubeEnv(env.SONARQUBE_SERVER) {
                    sh 'mvn -B sonar:sonar'
                }
            }
        }

        stage('OWASP Dependency Check') {
            when {
                expression { env.OWASP_ENABLED == 'true' }
            }
            steps {
                // Requires "OWASP Dependency-Check" plugin and a configured installation
                dependencyCheck additionalArguments: '--format XML --scan .', odcInstallation: 'Default'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Docker Build & Tag') {
            steps {
                sh """
                echo "Building Docker image..."
                docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                docker images | grep ${IMAGE_NAME}
                """
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                sh """
                echo "=== Trivy scan (report only) ==="
                docker run --rm \\
                  -v /var/run/docker.sock:/var/run/docker.sock \\
                  aquasec/trivy image ${IMAGE_NAME}:${BUILD_NUMBER} || true

                if [ "${TRIVY_FAIL_ON_HIGH}" = "true" ]; then
                  echo "=== Trivy gate: fail on HIGH/CRITICAL ==="
                  docker run --rm \\
                    -v /var/run/docker.sock:/var/run/docker.sock \\
                    aquasec/trivy image \\
                      --severity HIGH,CRITICAL \\
                      --ignore-unfixed \\
                      --exit-code 1 \\
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                else
                  echo "Skipping failure gate on HIGH/CRITICAL (TRIVY_FAIL_ON_HIGH=false)"
                fi
                """
            }
        }

        stage('Kubernetes Deploy + Rollout') {
            steps {
                withEnv(["KUBECONFIG=${KUBECONFIG_PATH}"]) {
                    sh """
                    echo "Applying Kubernetes manifests..."
                    kubectl apply -n ${K8S_NAMESPACE} -f k8s/deployment.yaml
                    kubectl apply -n ${K8S_NAMESPACE} -f k8s/service.yaml

                    echo "Waiting for rollout of deployment/${K8S_DEPLOYMENT}..."
                    kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --timeout=60s

                    echo "Current pods:"
                    kubectl get pods -n ${K8S_NAMESPACE} -o wide

                    echo "Service info:"
                    kubectl get svc -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline SUCCESS: App should be available at http://localhost:${SERVICE_PORT}"
        }
        failure {
            echo "❌ Pipeline FAILED: check stage logs above."
        }
    }
}
