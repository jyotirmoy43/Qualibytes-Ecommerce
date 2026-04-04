
pipeline {
    agent any

    environment {
        DOCKER_USER = 'jyotirmoy43'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        KUBE_CONFIG = '/home/jenkins/.kube/config'
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Clone Repository') {
            steps {
                sh '''
                    rm -rf ecom || true
                    git clone -b main https://github.com/jyotirmoy43/Qualibytes-Ecommerce.git ecom
                '''
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
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
                                docker build -t ${DOCKER_USER}/qbshop-app:${IMAGE_TAG} .
                            '''
                        }
                    }
                }

                stage('Build Migration Image') {
                    steps {
                        dir('ecom') {
                            sh '''
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
                        echo "Updating image tags..."

                        # Fix REPLACE_TAG
                        sed -i "s|REPLACE_TAG|${IMAGE_TAG}|g" kubernetes/08-qbshop-deployment.yaml
                        sed -i "s|REPLACE_TAG|${IMAGE_TAG}|g" kubernetes/12-migration-job.yaml

                        echo "Applying Kubernetes manifests..."

                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/01-namespace.yaml
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/04-configmap.yaml
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/05-secrets.yaml

                        # MongoDB
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/02-mongodb-pv.yaml || true
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/03-mongodb-pvc.yaml

                        # 🔥 FIX: delete service before reapply
                        kubectl --kubeconfig=${KUBE_CONFIG} delete service mongodb-service -n qbshop || true
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/06-mongodb-service.yaml

                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/07-mongodb-statefulset.yaml

                        # App
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/08-qbshop-deployment.yaml
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/09-qbshop-service.yaml

                        # Migration
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/12-migration-job.yaml || true

                        # Optional
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/10-ingress.yaml || true
                        kubectl --kubeconfig=${KUBE_CONFIG} apply -f kubernetes/11-hpa.yaml || true

                        echo "Restarting deployment..."
                        kubectl --kubeconfig=${KUBE_CONFIG} rollout restart deployment qbshop -n qbshop || true
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Pods:"
                    kubectl --kubeconfig=${KUBE_CONFIG} get pods -n qbshop

                    echo "Services:"
                    kubectl --kubeconfig=${KUBE_CONFIG} get svc -n qbshop
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully! Image tag: ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}      
