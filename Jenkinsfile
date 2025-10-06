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
                    // Ensure the kubeconfig file path is properly passed to kubeaudit
                    withKubeConfig(
                        credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}",
                        clusterName: "${env.K8S_CLUSTER}",
                        namespace: "${env.K8S_NAMESPACE}",
                        serverUrl: "${env.K8S_SERVER}"
                    ) {
                        // Set the KUBECONFIG environment variable for the current shell session
                        withEnv(["KUBECONFIG=${env.KUBECONFIG_CREDENTIALS_ID}"]) {
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
