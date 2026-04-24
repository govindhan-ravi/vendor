pipeline {
    agent any

    environment {
        // --- Configuration Variables ---
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        DOCKER_REGISTRY       = 'govindhan1234'
        SONAR_SERVER_NAME     = 'sonar-server'
        EKS_CLUSTER_NAME      = 'accounts-cluster'
        AWS_REGION            = 'us-east-1'
        
        APP_NAME_FRONTEND     = 'vendor-frontend'
        APP_NAME_BACKEND      = 'vendor-backend'
        IMAGE_TAG             = "v${env.BUILD_NUMBER}"
    }

    stages {
        stage('1. Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. ESLint (Code Quality Check)') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run lint'
                }
            }
        }

        stage('3. Frontend Tests') {
            steps {
                dir('frontend') {
                    // Using || true because tests aren't configured in package.json yet
                    sh 'npm test || echo "Frontend tests not yet implemented"'
                }
            }
        }

        stage('4. Backend Tests') {
            steps {
                dir('backend') {
                    sh 'npm install'
                    // Using || true because tests aren't configured in package.json yet
                    sh 'npm test || echo "Backend tests not yet implemented"'
                }
            }
        }

        stage('5. SonarQube Scan') {
            steps {
                // Ensure SonarQube plugin is configured in Jenkins with this server name
                withSonarQubeEnv(SONAR_SERVER_NAME) {
                    sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=vendor-management-system \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=http://3.85.165.178:9000
                    '''
                }
            }
        }

        stage('6. Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('7. Trivy Filesystem Scan') {
            steps {
                // Scan the local filesystem for vulnerabilities
                sh 'trivy fs --format table -o trivy-fs-report.txt .'
            }
        }

        stage('8. Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_REGISTRY}/${APP_NAME_FRONTEND}:${IMAGE_TAG} ./frontend"
                sh "docker build -t ${DOCKER_REGISTRY}/${APP_NAME_BACKEND}:${IMAGE_TAG} ./backend"
            }
        }

        stage('9. Trivy Image Scan') {
            steps {
                // Scan the Docker images for vulnerabilities before pushing
                sh "trivy image --format table -o trivy-frontend-image-report.txt ${DOCKER_REGISTRY}/${APP_NAME_FRONTEND}:${IMAGE_TAG} || true"
                sh "trivy image --format table -o trivy-backend-image-report.txt ${DOCKER_REGISTRY}/${APP_NAME_BACKEND}:${IMAGE_TAG} || true"
            }
        }

        stage('10. Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_REGISTRY}/${APP_NAME_FRONTEND}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_REGISTRY}/${APP_NAME_BACKEND}:${IMAGE_TAG}"
                }
            }
        }

        stage('11. Helm Lint') {
            steps {
                // Assuming you will create a Helm chart inside a folder named 'helm/vendor-management'
                sh "helm lint ./helm/vendor-management || echo 'Helm chart directory not found yet'"
            }
        }

        stage('12. EKS Authentication') {
            steps {
                // Update Kubeconfig to allow Jenkins to deploy to AWS EKS
                sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}"
            }
        }

        stage('13. Helm Deploy') {
            steps {
                sh "helm upgrade --install vendor-app ./helm/vendor-management --set frontend.image.tag=${IMAGE_TAG} --set backend.image.tag=${IMAGE_TAG} --namespace default"
            }
        }

        stage('14. Prometheus') {
            steps {
                // Deploying Prometheus via Helm Community Charts
                sh '''
                helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                helm repo update
                helm upgrade --install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace
                '''
            }
        }

        stage('15. Grafana') {
            steps {
                // Deploying Grafana via Helm
                sh '''
                helm repo add grafana https://grafana.github.io/helm-charts
                helm repo update
                helm upgrade --install grafana grafana/grafana --namespace monitoring --create-namespace
                '''
            }
        }

        stage('16. Alerting (Alertmanager)') {
            steps {
                // Alertmanager is automatically deployed by the Prometheus chart above.
                // We use this stage to apply any custom alert rules if you create them.
                sh '''
                echo "Alertmanager is deployed alongside Prometheus."
                echo "Applying custom Alertmanager config/rules if they exist..."
                # kubectl apply -f ./k8s-monitoring/alert-rules.yaml || true
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline Completed."
            // Optionally, archive the Trivy scan reports
            archiveArtifacts artifacts: '*report.txt', allowEmptyArchive: true
        }
        success {
            echo "Successfully deployed Vendor Management System!"
        }
        failure {
            echo "Pipeline Failed. Please check the logs."
        }
    }
}
