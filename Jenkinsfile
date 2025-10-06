pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'M2_HOME'
    }

    environment {
        IMAGE_NAME = 'hadilsae/hadilpfe'
        IMAGE_TAG = 'latest'
        TRIVY_CACHE_DIR = '/var/lib/jenkins/.cache/trivy'
        
        // Kubernetes deployment environment variables
        KUBECONFIG_CREDENTIALS_ID = 'k8-cred'
        K8S_CLUSTER = 'kubernetes'
        K8S_NAMESPACE = 'webapps'
        K8S_SERVER = 'https://192.168.1.18:6443'
        
        // Kubeaudit
        KUBEAUDIT_VERSION = '0.22.1'
    }

    stages {
        stage('Install Kubeaudit') {
            steps {
                script {
                    def kubeauditExists = sh(script: 'command -v kubeaudit', returnStatus: true) == 0
                    
                    if (!kubeauditExists) {
                        echo 'Installing kubeaudit...'
                        sh """
                            wget -q https://github.com/Shopify/kubeaudit/releases/download/v${KUBEAUDIT_VERSION}/kubeaudit_${KUBEAUDIT_VERSION}_linux_amd64.tar.gz
                            tar -xzf kubeaudit_${KUBEAUDIT_VERSION}_linux_amd64.tar.gz
                            chmod +x kubeaudit
                            sudo mv kubeaudit /usr/local/bin/ || mv kubeaudit ${WORKSPACE}/
                        """
                    } else {
                        echo 'Kubeaudit already installed'
                    }
                }
            }
        }

        stage('Checkout Git') {
            steps {
                git url: 'https://github.com/Hadilsa/applicatioon-pfe.git', branch: 'main'
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

        stage('Update Maven Version') {
            steps {
                script {
                    def newVersion = "0.0.${env.BUILD_NUMBER}"
                    echo "Setting Maven version to ${newVersion}"
                    sh "mvn versions:set -DnewVersion=${newVersion} -DgenerateBackupPoms=false"
                }
            }
        }

        stage('Build (Maven Package)') {
            steps {
                sh 'mvn clean verify -P test-coverage'
            }
        }

        stage('Publish JaCoCo Report') {
            steps {
                publishHTML(target: [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'target/site/jacoco',
                    reportFiles: 'index.html',
                    reportName: 'JaCoCo Code Coverage Report'
                ])
            }
        }

        stage('Déploiement Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-settings',
                    jdk: 'jdk17',
                    maven: 'M2_HOME'
                ) {
                    sh 'mvn deploy -DskipTests'
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
                script {
                    sh "trivy image --cache-dir ${TRIVY_CACHE_DIR} --timeout 10m --format table -o trivy-image-report.html ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}",
                    clusterName: "${env.K8S_CLUSTER}",
                    namespace: "${env.K8S_NAMESPACE}",
                    serverUrl: "${env.K8S_SERVER}"
                ) {
                    sh "kubectl apply -f deployment-service.yaml -n ${env.K8S_NAMESPACE}"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}",
                    clusterName: "${env.K8S_CLUSTER}",
                    namespace: "${env.K8S_NAMESPACE}",
                    serverUrl: "${env.K8S_SERVER}"
                ) {
                    sh "kubectl get pods -n ${env.K8S_NAMESPACE}"
                    sh "kubectl get svc -n ${env.K8S_NAMESPACE}"
                }
            }
        }

        stage('Kubeaudit Security Scan') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'k8-cred', variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            export PATH=\$PATH:${WORKSPACE}
                            
                            echo "=== Starting Kubeaudit Security Scan ==="
                            echo "Namespace: ${env.K8S_NAMESPACE}"
                            
                            # Scan deployed resources in the cluster
                            kubeaudit all \
                                --kubeconfig \${KUBECONFIG} \
                                --namespace ${env.K8S_NAMESPACE} \
                                --format pretty > kubeaudit-report.txt 2>&1 || true
                            
                            echo "=== Kubeaudit Scan Results ===" 
                            cat kubeaudit-report.txt
                        """
                        
                        // Analyze results
                        def issuesFound = sh(
                            script: 'grep -c "\\[error\\]\\|\\[warning\\]" kubeaudit-report.txt || true',
                            returnStdout: true
                        ).trim()
                        
                        def issueCount = 0
                        try {
                            issueCount = issuesFound.toInteger()
                        } catch (Exception e) {
                            echo "Could not parse issue count, defaulting to 0"
                        }
                        
                        if (issueCount > 0) {
                            echo "⚠️ Found ${issueCount} security issues - Please review the report"
                        } else {
                            echo "✅ No security issues found"
                        }
                    }
                }
            }
        }

        stage('Archive Security Reports') {
            steps {
                // Archive Trivy reports
                archiveArtifacts artifacts: 'trivy-*.html', allowEmptyArchive: true
                
                // Archive Kubeaudit report
                archiveArtifacts artifacts: 'kubeaudit-report.txt', allowEmptyArchive: true
                
                // Generate HTML report for Kubeaudit
                script {
                    if (fileExists('kubeaudit-report.txt')) {
                        sh """
                            echo '<!DOCTYPE html><html><head><title>Kubeaudit Security Report</title>' > kubeaudit-report.html
                            echo '<style>body{font-family:monospace;background:#1e1e1e;color:#d4d4d4;padding:20px;}pre{white-space:pre-wrap;}</style>' >> kubeaudit-report.html
                            echo '</head><body><h1>Kubeaudit Security Scan Report</h1><pre>' >> kubeaudit-report.html
                            cat kubeaudit-report.txt >> kubeaudit-report.html
                            echo '</pre></body></html>' >> kubeaudit-report.html
                        """
                        
                        publishHTML([
                            allowMissing: true,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: '.',
                            reportFiles: 'kubeaudit-report.html',
                            reportName: 'Kubeaudit Security Report'
                        ])
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
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
        <p>Security reports attached:</p>
        <ul>
            <li>Trivy File System Scan</li>
            <li>Trivy Docker Image Scan</li>
            <li>Kubeaudit Kubernetes Security Scan</li>
        </ul>
    </div>
</body>
</html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'hadil.saidi16@gmail.com',
                    from: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html,kubeaudit-report.txt'
                )

                archiveArtifacts 'target/site/jacoco/**'
                
                // Clean up
                sh 'rm -f kubeaudit_*.tar.gz'
            }
        }

        success {
            echo '✅ Pipeline completed successfully!'
        }

        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
