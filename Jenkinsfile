pipeline {
    agent any

    environment {
        IMAGE_NAME = 'hadilsae/hadilpfe'
        IMAGE_TAG = 'latest'
        TRIVY_CACHE = '/var/lib/jenkins/.cache/trivy'  // Use persistent cache
    }

    stages {
        stage('Build & Push Docker Image') {
            steps {
                script {
                    sh 'docker version'

                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker info'
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Docker Image Scan (Trivy)') {
            steps {
                sh """
                    trivy image \
                    --cache-dir ${TRIVY_CACHE} \
                    --timeout 10m \
                    --format table \
                    -o trivy-image-report.html \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }
}
