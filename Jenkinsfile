pipeline {
    agent any

    environment {
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
                    // Use the `withCredentials` block to inject the kubeconfig file
                    withCredentials([file(credentialsId: 'k8-cred', variable: 'KUBECONFIG_FILE')]) {
                        // Ensure the KUBECONFIG environment variable is set to the correct path
                        withEnv(["KUBECONFIG=${KUBECONFIG_FILE}"]) {
                            // Run the kubeaudit scan, check all resources (pods, deployments, services, etc.)
                            sh "kubeaudit all --kubeconfig ${KUBECONFIG} --namespace ${env.K8S_NAMESPACE} --format pretty"
                        }
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
        failure {
            echo '❌ Kubeaudit Scan failed!'
        }
    }
}
