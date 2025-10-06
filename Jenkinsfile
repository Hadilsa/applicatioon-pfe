pipeline {
    agent any

    environment {
        KUBECONFIG_CREDENTIALS_ID = 'k8-cred'
        K8S_CLUSTER = 'kubernetes'
        K8S_NAMESPACE = 'webapps'
        K8S_SERVER = 'https://192.168.1.18:6443'
    }

    stages {
        stage('Kubeaudit Scan') {
            steps {
                withKubeConfig(
                    credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}",
                    clusterName: "${env.K8S_CLUSTER}",
                    namespace: "${env.K8S_NAMESPACE}",
                    serverUrl: "${env.K8S_SERVER}"
                ) {
                    sh '''
                        kubeaudit all --live > kubeaudit-report.txt || true
                        cat kubeaudit-report.txt
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts 'kubeaudit-report.txt'
        }
    }
}
