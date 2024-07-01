pipeline {
    agent {
        docker {
            image 'docker:19.03.12'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_IMAGE = 'kwonhyeokjun/spring-petclinic'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials-id'
        GIT_REPO_URL = 'https://github.com/K-hyeokjun/spring-petclinic-docker'
        GIT_BRANCH = 'main'
        KUBECONFIG_CREDENTIAL_ID = 'your-kubeconfig-credentials-id'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}"
            }
        }

        stage('Build') {
            steps {
                sh './mvnw clean package'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE}:${env.BUILD_ID}", "--cache-from=${DOCKER_IMAGE}:latest .")
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    docker.withRegistry('', "${DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push()
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f k8s/mysql-config-persistentvolumeclaim.yaml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl apply -f k8s/mysql-data-persistentvolumeclaim.yaml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl apply -f k8s/mysql-deployment.yaml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl apply -f k8s/mysql-service.yaml --kubeconfig=$KUBECONFIG'
                    }
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    // Update Kubernetes deployment YAML file with new Docker image tag
                    sh 'sed -i "34s|.*|          image: ${DOCKER_IMAGE}:${env.BUILD_ID}|" k8s/petclinic-deployment.yaml'
                    
                    // Commit and push the changes
                    sh 'git config user.email "gurwns4643@gmail.com"'
                    sh 'git config user.name "hyeokjun Kwon"'
                    sh 'git add k8s/petclinic-deployment.yaml'
                    sh 'git commit -m "Update image to ${DOCKER_IMAGE}:${env.BUILD_ID}"'
                    sh 'git push'
                }
            }
        }

        stage('Deploy PetClinic') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f k8s/petclinic-deployment.yaml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl apply -f k8s/petclinic-service.yaml --kubeconfig=$KUBECONFIG'
                    }
                }
            }
        }

        stage('Sync with Argo CD') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        // Sync the application using kubectl
                        sh 'kubectl apply -f k8s/petclinic-deployment.yaml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl rollout status deployment/petclinic -n devops-tools --kubeconfig=$KUBECONFIG'
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'The build and deployment were successful!'
        }
        failure {
            echo 'The build or deployment failed.'
        }
    }
}
