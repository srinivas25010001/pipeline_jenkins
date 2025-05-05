pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'docker-hub-creds'
        IMAGE_NAME = 'srinivas0001/jute_8'
        GITHUB_CREDENTIALS = 'github-creds'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', credentialsId: GITHUB_CREDENTIALS, url: 'https://github.com/srinivas25010001/dev_jute_smart.git'
            }
        }

        stage('Clone frappe_docker') {
            steps {
                sh 'rm -rf frappe_docker || true'
                sh 'git clone https://github.com/frappe/frappe_docker'
            }
        }

        stage('Create apps.json') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        def appsJsonContent = """
                        [
                            {
                                "url": "https://github.com/frappe/erpnext",
                                "branch": "version-15"
                            },
                            {
                                "url": "https://${GIT_USER}:${GIT_PASS}@github.com/srinivas25010001/dev_jute_smart.git",
                                "branch": "main"
                            }
                        ]
                        """
                        writeFile(file: 'apps.json', text: appsJsonContent)
                    }
                }
            }
        }

        stage('Encode apps.json') {
            steps {
                script {
                    sh "base64 -w 0 apps.json > apps.json.b64"
                }
            }
        }

        stage('Build Docker Image') {
    steps {
        script {
            def appsJsonBase64 = sh(script: "cat apps.json.b64", returnStdout: true).trim()
            dir('frappe_docker') {
                sh """
                    sudo docker buildx build \
                      --platform=linux/amd64 \
                      --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
                      --build-arg=FRAPPE_BRANCH=version-15 \
                      --build-arg=PYTHON_VERSION=3.11.6 \
                      --build-arg=NODE_VERSION=18.18.2 \
                      --build-arg=APPS_JSON_BASE64=${appsJsonBase64} \
                      --tag=${IMAGE_NAME}:latest \
                      --file=images/custom/Containerfile \
                """
            }
        }
    }
}


        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }
    }

    post {
        success {
            echo "Build and push completed successfully!"
        }
        failure {
            echo "Build failed!"
        }
    }
}
