pipeline {
    agent any

    environment {
        DOCKER_USER = 'jyotirmoy43'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        KUBE_CONFIG = '/home/jenkins/.kube/config'       // Adjust if needed
        DEPLOY_FILE = 'kubernetes/deployment.yaml'      // Adjust if needed
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Clone Repository') {
            steps {
                // Explicit git clone to avoid "checkout scm" errors
                sh '''
                    if [ -d ecom ]; then
                        rm -rf ecom
                    fi
                    git clone -b main https://github.com/jyotirmoy43/Qualibytes-Ecommerce.git ecom
                '''
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "Checking for required tools..."
                    command -v docker >/dev/null || { echo "Docker not installed"; exit 1; }
                    command -v kubectl >/dev/null || { echo "kubectl not installed"; exit 1; }
                '''
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build App Image') {
                    steps {
                        dir('ecom') {
                            sh '''
                                echo "Building main app image..."
                                docker build -t ${DOCKER_USER}/qbshop-app:${IMAGE_TAG} .
                            '''
                        }
                    }
                }

                stage('Build Migration Image') {
                    steps {
                        dir('ecom') {
                            sh '''
                                echo "Building migration image..."
                                docker build -t ${DOCKER_USER}/qbshop-migration:${IMAGE_TAG} -f scripts/Dockerfile.migration .
                            '''
                        }
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-password', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u ${DOCKER_USER} --password-stdin
                        docker push ${DOCKER_USER}/qbshop-app:${IMAGE_TAG}
                        docker push ${DOCKER_USER}/qbshop-migration:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes (EKS)') {
            steps {
                dir('ecom') {
                    sh '''
                        if [ ! -f ${DEPLOY_FILE} ]; then
                            echo "Deployment file ${DEPLOY_FILE} not found!"
                            exit 1
                        fi

                        echo "Updating Kubernetes deployment..."
                        sed -i "s|REPLACE_TAG|${IMAGE_TAG}|g" ${DEPLOY_FILE}
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f ${DEPLOY_FILE}
                    '''
                }
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
