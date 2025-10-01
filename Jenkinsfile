pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'M2_HOME'  // Make sure this matches your Jenkins Maven tool name
    }

    environment {
        IMAGE_NAME = 'hadilsae/hadilpfe'   // Docker Hub repo
        IMAGE_TAG = 'latest'                // Image tag
    }

    stages {
        stage('Checkout Git') {
            steps {
                git url: 'https://github.com/Hadilsa/application-pfe.git', branch: 'main'  // fixed typo
            }
        }

        stage('Maven Clean') {
            steps {
                sh 'mvn clean'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan (Trivy)') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
mvn sonar:sonar \
-Dsonar.projectKey=BoardGame \
-Dsonar.projectName=BoardGame \
-Dsonar.login=admin \
-Dsonar.password=0000
                    '''.stripIndent()
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube_key2'
                }
            }
        }

        stage('Build (Maven Package)') {
            steps {
                sh 'mvn package -Dmaven.test.skip=true -P test-coverage'
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-settings',
                    jdk: 'jdk17',
                    maven: 'M2_HOME',
                    mavenSettingsConfig: '',
                    traceability: true
                ) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
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

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'kubernetes',
                    namespace: 'webapps',
                    serverUrl: 'https://192.168.1.18:6443'
                ) {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'kubernetes',
                    namespace: 'webapps',
                    serverUrl: 'https://192.168.1.18:6443'
                ) {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
<html>
<body>
    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
        <h3 style="color: ${bannerColor};">${pipelineStatus.toUpperCase()}</h3>
    </div>
    <h2>${jobName} - Build ${buildNumber}</h2>
    <div style="background-color: ${bannerColor}; padding: 10px;">
        <h3 style="color: white;">Pipeline Status</h3>
        <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
    </div>
</body>
</html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'hadil.saidi16@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }

        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
