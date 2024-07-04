pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub' // Docker Hub 자격 증명 ID
        GIT_REPO = 'https://github.com/K-hyeokjun/spring-petclinic-docker.git'
        BRANCH_NAME = 'main'
        ARGOCD_SERVER = 'a3823903901c24a01bcade663ec3457d-1327040958.ap-northeast-2.elb.amazonaws.com'
        ARGOCD_APP_NAME = 'spring-petclinic'
        DOCKER_IMAGE = 'kwonhyeokjun/myjenkins-blueocean' // Docker Hub 이미지 이름
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BRANCH_NAME}", url: "${GIT_REPO}"
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    def tagsOutput = sh(
                        script: "curl -s https://registry.hub.docker.com/v1/repositories/${DOCKER_IMAGE}/tags",
                        returnStdout: true
                    ).trim()

                    def tags = tagsOutput.readLines().collect { it.split(':')[1].replace('"', '').replace('}', '').trim() }
                    echo "Existing tags in Docker Hub: ${tags}"

                    def latestTag = tags.collect { it.tokenize('.').collect { it.toInteger() } }.max { a, b -> 
                        for (i in 0..<Math.min(a.size(), b.size())) {
                            if (a[i] != b[i]) return a[i] <=> b[i]
                        }
                        return a.size() <=> b.size()
                    } ?: [1, 0, 0]

                    def (major, minor, patch) = latestTag

                    if (patch >= 15) {
                        patch = 0
                        minor += 1
                    } else {
                        patch += 1
                    }

                    if (minor >= 10) {
                        minor = 0
                        major += 1
                    }

                    env.NEW_VERSION = "${major}.${minor}.${patch}"
                    echo "New version: ${env.NEW_VERSION}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'echo $(date) > build_info.txt'
                sh './mvnw clean package -Dcheckstyle.skip=true'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_CREDENTIALS_ID) {
                        def app = docker.build("${DOCKER_IMAGE}:${env.NEW_VERSION}")
                        app.push()
                        app.push('latest')
                    }

                    // Update the Kubernetes deployment file
                    if (fileExists('k8s/petclinic-deployment.yaml')) {
                        sh """
                        sed -i 's#image: ${DOCKER_IMAGE}:.*#image: ${DOCKER_IMAGE}:${env.NEW_VERSION}#' k8s/petclinic-deployment.yaml
                        """
                    } else {
                        error "k8s/petclinic-deployment.yaml file not found"
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'argocd-creds', usernameVariable: 'ARGOCD_USERNAME', passwordVariable: 'ARGOCD_PASSWORD')]) {
                        sh '''
                        argocd login ${ARGOCD_SERVER} --username ${ARGOCD_USERNAME} --password ${ARGOCD_PASSWORD} --insecure --grpc-web
                        argocd app sync ${ARGOCD_APP_NAME}
                        '''
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
        /*
        always {
            script {
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh 'git config --global user.email "gurwns4643@gmail.com"'
                    sh 'git config --global user.name "K-hyeokjun"'
                    sh 'git pull origin ${BRANCH_NAME}'

                    def changes = sh(script: 'git status --porcelain', returnStdout: true).trim()
                    if (changes) {
                        sh 'git add k8s/petclinic-deployment.yaml'
                        sh "git commit -m 'Update deployment to version ${env.NEW_VERSION}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/K-hyeokjun/spring-petclinic-docker.git ${BRANCH_NAME}"
                    } else {
                        echo "No changes to commit"
                    }
                }
            }
        }
        */
    }
}
