pipeline {
    agent any

    environment {
        KUBECONFIG_CREDENTIALS_ID = 'k8-cred'  // Jenkins credentials for kubeconfig
        K8S_CLUSTER = 'kubernetes'
        K8S_NAMESPACE = 'webapps'
        K8S_SERVER = 'https://192.168.1.18:6443'  // Adjust as needed
    }

    stages {
        stage('Checkout Git') {
            steps {
                // Checkout the Kubernetes manifests or your repo
                git url: 'https://github.com/Hadilsa/applicatioon-pfe.git', branch: 'main'
            }
        }

        stage('Run Kubeaudit Scan') {
            steps {
                script {
                    // Ensure kubeaudit is installed on the Jenkins agent/worker

                    // Run kubeaudit to check the Kubernetes manifests
                    withKubeConfig(
                        credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}",
                        clusterName: "${env.K8S_CLUSTER}",
                        namespace: "${env.K8S_NAMESPACE}",
                        serverUrl: "${env.K8S_SERVER}"
                    ) {
                        // Run the kubeaudit scan, check all resources (pods, deployments, services, etc.)
                        sh "kubeaudit all --kubeconfig ${KUBECONFIG_CREDENTIALS_ID} --namespace ${env.K8S_NAMESPACE} --format pretty"
                    }
                }
            }
        }

        stage('Archive Kubeaudit Report') {
            steps {
                // Archive the report for further inspection
                archiveArtifacts artifacts: 'kubeaudit-report.txt', allowEmptyArchive: false
            }
        }
    }

    post {
        always {
            // Send email or perform other actions after scan
            emailext(
                subject: "Kubeaudit Scan Report - Build ${env.BUILD_NUMBER}",
                body: "The Kubeaudit scan has completed for this build. Please review the attached report.",
                to: 'your-email@example.com',  // Adjust the recipient as needed
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'kubeaudit-report.txt'  // Ensure this file path matches your generated report
            )
        }

        failure {
            echo '❌ Kubeaudit Scan failed!'
        }
    }
}
