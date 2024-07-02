pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'kwonhyeokjun/spring-petclinic'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials-id'
        GIT_REPO_URL = 'https://github.com/K-hyeokjun/spring-petclinic-docker'
        GIT_BRANCH = 'main'
        GIT_CREDENTIALS_ID = 'your-git-credentials-id'
        KUBECONFIG_CREDENTIAL_ID = 'your-kubeconfig-credentials-id'
    }

    stages {
        stage('Ensure Docker Permissions') {
            steps {
                script {
                    echo 'Ensuring Docker permissions...'
                    sh 'chown root:docker /var/run/docker.sock'
                    sh 'chmod 666 /var/run/docker.sock'
                }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    echo 'Checking out code from Git...'
                    git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}", credentialsId: "${GIT_CREDENTIALS_ID}"
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    echo 'Building the project...'
                    sh './mvnw clean package'
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo 'Building Docker image...'
                    dockerImage = docker.build("${DOCKER_IMAGE}:${env.BUILD_ID}")
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    echo 'Pushing Docker image to registry...'
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
                    echo 'Deploying MySQL...'
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
                    echo 'Updating Kubernetes manifests...'
                    sh 'sed -i "34s|.*|          image: ${DOCKER_IMAGE}:${env.BUILD_ID}|" k8s/petclinic-deployment.yaml'
                    sh 'git config user.email "gurwns4643@gmail.com"'
                    sh 'git config user.name "hyeokjun Kwon"'
                    sh 'git add k8s/petclinic-deployment.yaml'
                    sh 'git commit -m "Update image to ${DOCKER
