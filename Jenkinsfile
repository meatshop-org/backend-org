pipeline {
    agent any
    stages {
        stage('Install Dependencies in venv') {
            steps {
                sh '''
                    python3.11 -m venv venv
                    . venv/bin/activate
                    python -m pip install --upgrade pip
                    pip install -r requirements.txt
                    python3.11 -m pip install pip-audit
                '''
            }
        }

        stage('Audit Dependencies') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip-audit > pip-audit-audit.txt
                '''

                script {
                    def auditReport = readFile('pip-audit-report.txt')
                    if (auditReport.contains('Found')) {
                        error 'Found vulnerabilities in dependencies!'
                    }
                }
            }
        }

        post {
            always {
                archiveArtifacts allowEmptyArchive: true, artifacts: 'pip-audit-report.txt', followSymlinks: false
            }
        }
}
