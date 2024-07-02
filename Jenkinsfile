pipeline {
    agent {
        docker {
            image 'docker:19.03.12'
            args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
            label 'jenkins-node' // 추가된 라벨 사용
        }
    }

    environment {
        DOCKER_IMAGE = 'kwonhyeokjun/spring-petclinic'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials-id'
        GIT_REPO_URL = 'https://github.com/K-hyeokjun/spring-petclinic-docker'
        GIT_BRANCH = 'main'
        GIT_CREDENTIALS_ID = 'your-git-credentials-id'
        KUBECONFIG_CREDENTIAL_ID = 'your-kubeconfig-credentials-id'
        DOCKERHUB_USERNAME = 'kwonhyeokjun'
        DOCKERHUB_PASSWORD = 'kwon1715!'
    }

    stages {
        stage('Ensure Docker Permissions') {
            steps {
                script {
                    sh 'chown root:docker /var/run/docker.sock'
                    sh 'chmod 660 /var/run/docker.sock'
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
                    sh 'docker build -t ${DOCKER_IMAGE}:${env.BUILD_ID} .'
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    echo 'Pushing Docker image to registry...'
                    sh 'docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_PASSWORD'
                    sh 'docker push ${DOCKER_IMAGE}:${env.BUILD_ID}'
                    sh 'docker push ${DOCKER_IMAGE}:latest'
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
                    sh 'git commit -m "Update image to ${DOCKER_IMAGE}:${env.BUILD_ID}"'
                    sh 'git push'
                }
            }
        }

        stage('Deploy PetClinic') {
            steps {
                script {
                    echo 'Deploying PetClinic application...'
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
                    echo 'Syncing with Argo CD...'
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f k8s/petclinic-deployment.yaml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl rollout status deployment/petclinic -n devops-tools --kubeconfig=$KUBECONFIG'
                    }
                }
            }
        }
    }

    post {
        always {
            node('jenkins-node') {
                cleanWs()
            }
        }
        success {
            echo 'The build and deployment were successful!'
        }
        failure {
            echo 'The build or deployment failed.'
        }
    }
}
