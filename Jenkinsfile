pipeline {
    agent any

    tools {
        jdk 'jdk17'         // Adjust if your Jenkins uses a different JDK name
        maven 'M2_HOME'     // Must match your Jenkins Maven tool name
    }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/Hadilsa/applicatioon-pfe.git', branch: 'main'
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(
                    maven: 'M2_HOME',
                    jdk: 'jdk17',
                    globalMavenSettingsConfig: 'global-settings'  // Make sure this config exists in Jenkins
                ) {
                    sh 'mvn deploy -DskipTests'
                }
            }
        }
    }

    post {
        success {
            echo '✅ Published to Nexus successfully!'
        }
        failure {
            echo '❌ Failed to publish to Nexus.'
        }
    }
}
