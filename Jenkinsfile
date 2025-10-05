pipeline {
    agent any

    environment {
        IMAGE_NAME = 'hadilsae/hadilpfe'
        IMAGE_TAG = 'latest'
    }

    stages {
        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Check docker client version
                    sh 'docker version'

                    // Login to Docker registry
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        // Show docker daemon info
                        sh 'docker info'

                        // Build docker image
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."

                        // Push docker image
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Docker Image Scan (Trivy)') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }
}
