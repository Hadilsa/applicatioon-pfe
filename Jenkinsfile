pipeline {
    agent any
    
    stages {
        stage('Test') {
            steps {
                echo '✅ Pipeline en cours...'
            }
        }
    }
    
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def statusColor = buildStatus == 'SUCCESS' ? 'green' : 'red'
                
                def emailBody = """
                <html>
                <body>
                    <div style="border: 4px solid ${statusColor}; padding: 20px;">
                        <h2 style="color: ${statusColor};">${buildStatus}</h2>
                        <h3>Job: ${jobName}</h3>
                        <h3>Build: #${buildNumber}</h3>
                        <p>Date: ${new Date()}</p>
                    </div>
                </body>
                </html>
                """
                
                emailext(
                    subject: "${jobName} - Build #${buildNumber} - ${buildStatus}",
                    body: emailBody,
                    to: 'hadil.saidi16@gmail.com',
                    from: 'jenkins@test.com',
                    mimeType: 'text/html'
                )
                
                echo '📧 Email envoyé !'
            }
        }
    }
}
