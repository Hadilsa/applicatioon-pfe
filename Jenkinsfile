pipeline {
    agent any
    
    environment {
        K8S_CLUSTER = 'kubernetes'
        K8S_NAMESPACE = 'webapps'
        K8S_SERVER = 'https://192.168.1.18:6443'
        KUBEAUDIT_VERSION = '0.22.1'  // Specify version for consistency
    }
    
    stages {
        stage('Install Kubeaudit') {
            steps {
                script {
                    // Check if kubeaudit is already installed
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
        
        stage('Run Kubeaudit Scan on Cluster') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'k8-cred', variable: 'KUBECONFIG_FILE')]) {
                        // Option 1: Scan live cluster resources
                        sh """
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            
                            # Scan all resources in the namespace
                            kubeaudit all \
                                --kubeconfig \${KUBECONFIG} \
                                --namespace ${env.K8S_NAMESPACE} \
                                --format pretty > kubeaudit-report.txt 2>&1 || true
                            
                            echo "=== Kubeaudit Scan Results ===" 
                            cat kubeaudit-report.txt
                        """
                    }
                }
            }
        }
        
        stage('Run Kubeaudit Scan on Manifests') {
            when {
                expression { 
                    // Run this stage if there are manifest files in the repo
                    fileExists('*.yaml') || fileExists('*.yml') || fileExists('k8s/')
                }
            }
            steps {
                script {
                    // Option 2: Scan manifest files in the repository
                    sh """
                        # Scan all YAML manifests in the repository
                        if [ -d "k8s" ]; then
                            kubeaudit all -f k8s/ --format pretty >> kubeaudit-manifest-report.txt 2>&1 || true
                        else
                            find . -name "*.yaml" -o -name "*.yml" | while read file; do
                                echo "Scanning \$file..." >> kubeaudit-manifest-report.txt
                                kubeaudit all -f "\$file" --format pretty >> kubeaudit-manifest-report.txt 2>&1 || true
                            done
                        fi
                        
                        if [ -f kubeaudit-manifest-report.txt ]; then
                            echo "=== Manifest Scan Results ==="
                            cat kubeaudit-manifest-report.txt
                        fi
                    """
                }
            }
        }
        
        stage('Analyze Results') {
            steps {
                script {
                    // Check if there are any security issues
                    def clusterIssues = sh(
                        script: 'grep -i "error\\|warn\\|fail" kubeaudit-report.txt || true',
                        returnStdout: true
                    ).trim()
                    
                    if (clusterIssues) {
                        echo "⚠️ Security issues found in cluster scan!"
                        echo "${clusterIssues}"
                        // Uncomment the next line to fail the pipeline on issues
                        // error("Kubeaudit found security issues")
                    } else {
                        echo "✅ No critical issues found in cluster scan"
                    }
                }
            }
        }
        
        stage('Archive Kubeaudit Reports') {
            steps {
                // Archive all reports
                archiveArtifacts artifacts: 'kubeaudit*.txt', allowEmptyArchive: true
                
                // Optional: Publish as HTML report
                script {
                    if (fileExists('kubeaudit-report.txt')) {
                        sh """
                            echo '<html><body><pre>' > kubeaudit-report.html
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
            // Clean up
            sh 'rm -f kubeaudit_*.tar.gz'
        }
        success {
            echo '✅ Kubeaudit Scan completed successfully!'
        }
        failure {
            echo '❌ Kubeaudit Scan failed!'
        }
    }
}
