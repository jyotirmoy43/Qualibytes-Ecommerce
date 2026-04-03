
   
pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = 'jyotirmoy43/qbshop-app'
        DOCKER_MIGRATION_IMAGE_NAME = 'jyotirmoy43/qbshop-migration'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        GIT_BRANCH = "dev"
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: "${GIT_BRANCH}", url: 'https://github.com/jyotirmoy43/Qualibytes-Ecommerce.git'
            }
        }

        stage('Cleanup Old Docker Resources') {
            steps {
                sh '''
                echo "Cleaning Docker system..."
                docker system prune -af || true
                '''
            }
        }

        stage('Build Docker Images') {
            parallel {

                stage('Build App Image') {
                    steps {
                        sh '''
                        echo "Building main app image..."
                        docker build -t $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG .
                        '''
                    }
                }

                stage('Build Migration Image') {
                    steps {
                        sh '''
                        echo "Building migration image..."
                        docker build -t $DOCKER_MIGRATION_IMAGE_NAME:$DOCKER_IMAGE_TAG -f scripts/Dockerfile.migration .
                        '''
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                echo "Running tests..."
                mvn test || true
                '''
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                sh '''
                echo "Running Trivy scan..."
                trivy image $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG || true
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo "Logging into DockerHub..."
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    echo "Pushing images..."
                    docker push $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG
                    docker push $DOCKER_MIGRATION_IMAGE_NAME:$DOCKER_IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes (EKS)') {
            steps {
                sh '''
                echo "Updating Kubernetes deployment..."

                # Update image tag in deployment file
                sed -i "s|image: .*|image: $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG|g" kubernetes/deployment.yaml

                # Apply to EKS cluster
                kubectl apply -f kubernetes/

                echo "Deployment updated successfully!"
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
