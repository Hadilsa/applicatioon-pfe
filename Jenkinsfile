pipeline {
    agent any

    environment {
        IMAGE_NAME = 'hadilsae/hadilpfe'
        IMAGE_TAG = 'latest'
        TRIVY_CACHE_DIR = '/var/lib/jenkins/.cache/trivy'
    }

    stages {
        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Check docker client version
                    sh 'docker version'

                    // Login and push to Docker Hub
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
                script {
                    sh "trivy image --cache-dir ${TRIVY_CACHE_DIR} --timeout 10m --format table -o trivy-image-report.html ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Archive Trivy Report') {
            steps {
                archiveArtifacts artifacts: 'trivy-image-report.html', allowEmptyArchive: false
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed.'
        }
    }
}
