pipeline {
    agent any

    environment {
        HARBOR_CREDENTIALS = 'harbor-creds' // Jenkins credentials ID for Harbor (username + password)
        IMAGE_NAME = '10.212.132.157/demo/test1' // Harbor image path
        GITHUB_CREDENTIALS = 'github-creds'
        FRAPPE_DOCKER_PATH = '/home/RAKPATE/frappe_docker'
    }

    stages {
        stage('Clone App Repository') {
            steps {
                git branch: 'main', credentialsId: GITHUB_CREDENTIALS, url: 'https://github.com/srinivas25010001/dev_jute_smart.git'
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
                sh "base64 -w 0 apps.json > apps.json.b64"
            }
        }

        stage('Login to Harbor') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDENTIALS, usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh """
                            echo "$HARBOR_PASS" | docker login 10.212.132.157 -u "$HARBOR_USER" --password-stdin
                            docker buildx create --use --name mybuilder || true
                            docker buildx inspect mybuilder --bootstrap
                        """
                    }
                }
            }
        }

        stage('Build and Push Multi-Arch Image') {
            steps {
                script {
                    def appsJsonBase64 = sh(script: "cat apps.json.b64", returnStdout: true).trim()
                    dir("${FRAPPE_DOCKER_PATH}") {
                        sh """
                            docker buildx use mybuilder
                            docker buildx build \
                              --platform=linux/amd64,linux/arm64 \
                              --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
                              --build-arg=FRAPPE_BRANCH=version-15 \
                              --build-arg=PYTHON_VERSION=3.11.6 \
                              --build-arg=NODE_VERSION=18.18.2 \
                              --build-arg=APPS_JSON_BASE64=${appsJsonBase64} \
                              --tag=${IMAGE_NAME}:latest \
                              --file=images/custom/Containerfile \
                              --push .
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Multi-arch Docker image built and pushed to Harbor successfully!"
        }
        failure {
            echo "❌ Build or push failed."
        }
    }
}
