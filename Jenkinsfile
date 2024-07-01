pipeline {
    agent any

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
                    dockerImage = docker.build("${DOCKER_IMAGE}:${env.BUILD_ID}")
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    docker.withRegistry('', "${DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push()
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
                    sh 'git config user.email "you@example.com"'
                    sh 'git config user.name "Your Name"'
                    sh 'git add k8s/petclinic-deployment.yaml'
                    sh 'git commit -m "Update image to ${DOCKER_IMAGE}:${env.BUILD_ID}"'
                    sh 'git push'
                }
            }
        }

        stage('Sync with Argo CD') {
            steps {
                script {
                    withKubeConfig([credentialsId: "${KUBECONFIG_CREDENTIAL_ID}"]) {
                        // Sync the application using kubectl
                        sh 'kubectl apply -f k8s/petclinic-deployment.yaml'
                        sh 'kubectl rollout status deployment/petclinic -n devops-tools'
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
