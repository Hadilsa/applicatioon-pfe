pipeline {
    agent any
    
    environment {
        K8S_NAMESPACE = 'webapps'
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
        
        stage('Run Kubeaudit Scan') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'k8-cred', variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            export PATH=\$PATH:${WORKSPACE}
                            
                            echo "=== Starting Kubeaudit Scan ==="
                            echo "Namespace: ${env.K8S_NAMESPACE}"
                            echo "Kubeconfig: \${KUBECONFIG}"
                            
                            # Run kubeaudit scan
                            kubeaudit all \
                                --kubeconfig \${KUBECONFIG} \
                                --namespace ${env.K8S_NAMESPACE} \
                                --format pretty > kubeaudit-report.txt 2>&1 || true
                            
                            echo "=== Kubeaudit Scan Results ===" 
                            cat kubeaudit-report.txt
                        """
                        
                        // Analyze results
                        def issuesFound = sh(
                            script: 'grep -c "Error\\|Warning" kubeaudit-report.txt || echo 0',
                            returnStdout: true
                        ).trim().toInteger()
                        
                        if (issuesFound > 0) {
                            echo "⚠️ Found ${issuesFound} security issues"
                            unstable(message: "Kubeaudit found ${issuesFound} security issues")
                        } else {
                            echo "✅ No security issues found"
                        }
                    }
                }
            }
        }
        
        stage('Archive Report') {
            steps {
                archiveArtifacts artifacts: 'kubeaudit-report.txt', allowEmptyArchive: false
                
                // Generate HTML report
                sh """
                    echo '<!DOCTYPE html><html><head><title>Kubeaudit Report</title>' > kubeaudit-report.html
                    echo '<style>body{font-family:monospace;background:#1e1e1e;color:#d4d4d4;padding:20px;}pre{white-space:pre-wrap;}</style>' >> kubeaudit-report.html
                    echo '</head><body><h1>Kubeaudit Security Scan Report</h1><pre>' >> kubeaudit-report.html
                    cat kubeaudit-report.txt >> kubeaudit-report.html
                    echo '</pre></body></html>' >> kubeaudit-report.html
                """
                
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'kubeaudit-report.html',
                    reportName: 'Kubeaudit Security Report'
                ])
            }
        }
    }
    
    post {
        always {
            sh 'rm -f kubeaudit_*.tar.gz'
        }
        success {
            echo '✅ Kubeaudit scan completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        unstable {
            echo '⚠️ Security issues detected - review the report'
        }
    }
}
