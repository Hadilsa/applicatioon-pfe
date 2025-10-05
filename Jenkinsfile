pipeline {
    agent any

    environment {
        KUBE_NAMESPACE = 'webapps'
        KUBE_SERVER_URL = 'https://192.168.1.18:6443'
        DEPLOYMENT_FILE = 'deployment-service.yaml'
    }

    stages {
        stage('Checkout Git') {
            steps {
                git url: 'https://github.com/Hadilsa/applicatioon-pfe.git', branch: 'main'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'kubernetes',
                    namespace: "${KUBE_NAMESPACE}",
                    serverUrl: "${KUBE_SERVER_URL}"
                ) {
                    sh "kubectl apply -f ${DEPLOYMENT_FILE} -n ${KUBE_NAMESPACE}"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'kubernetes',
                    namespace: "${KUBE_NAMESPACE}",
                    serverUrl: "${KUBE_SERVER_URL}"
                ) {
                    sh "kubectl get pods -n ${KUBE_NAMESPACE}"
                    sh "kubectl get svc -n ${KUBE_NAMESPACE}"
                }
            }
        }
    }

    post {
        failure {
            echo '❌ Deployment pipeline failed!'
        }
        success {
            echo '✅ Deployment pipeline succeeded!'
        }
    }
}
