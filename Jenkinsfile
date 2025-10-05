pipeline {
    agent any

    environment {
        // Optional: You can define global env variables here
        KUBECONFIG_CREDENTIALS_ID = 'k8-cred'
        K8S_CLUSTER = 'kubernetes'
        K8S_NAMESPACE = 'webapps'
        K8S_SERVER = 'https://192.168.1.18:6443'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                    sh 'kubectl apply -f deployment-service.yaml -n ${K8S_NAMESPACE}'
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
                    sh 'kubectl get pods -n ${K8S_NAMESPACE}'
                    sh 'kubectl get svc -n ${K8S_NAMESPACE}'
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Deployment failed!'
        }
    }
}
