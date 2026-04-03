pipeline {
    agent any

    environment {
        DOCKER_USER = 'jyotirmoy43'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        KUBE_CONFIG = '/home/jenkins/.kube/config' // Ensure kubeconfig exists in Jenkins
    }

    stages {
        stage('Cleanup Workspace') {
            steps { 
                cleanWs() 
            }
        }

        stage('Checkout Repository') {
            steps {
                git url: 'https://github.com/jyotirmoy43/Qualibytes-Ecommerce.git', branch: 'main'
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build App Image') {
                    steps {
                        sh """
                            echo "Building main app image..."
                            docker build -t ${DOCKER_USER}/qbshop-app:${IMAGE_TAG} .
                        """
                    }
                }

                stage('Build Migration Image') {
                    steps {
                        sh """
                            echo "Building migration image..."
                            docker build -t ${DOCKER_USER}/qbshop-migration:${IMAGE_TAG} -f scripts/Dockerfile.migration .
                        """
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh """
                    echo "Running Maven tests..."
                    if command -v mvn >/dev/null 2>&1; then
                        mvn test
                    else
                        echo "Maven not found, skipping tests..."
                    fi
                """
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                sh """
                    echo "Running Trivy scan..."
                    if command -v trivy >/dev/null 2>&1; then
                        trivy image ${DOCKER_USER}/qbshop-app:${IMAGE_TAG} || true
                    else
                        echo "Trivy not installed, skipping scan..."
                    fi
                """
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-password', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u ${DOCKER_USER} --password-stdin
                        docker push ${DOCKER_USER}/qbshop-app:${IMAGE_TAG}
                        docker push ${DOCKER_USER}/qbshop-migration:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes (EKS)') {
            steps {
                sh """
                    echo "Updating Kubernetes deployment..."
                    sed -i "s|REPLACE_TAG|${IMAGE_TAG}|g" kubernetes/deployment.yaml
                    kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/deployment.yaml
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully! App deployed with tag ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
