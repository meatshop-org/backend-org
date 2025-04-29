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
                    python3.11 -m pip install safety
                '''
            }
        }
        stage('Dependencies Scanning...'){
            parallel {
                stage('Audit Dependencies') {
                    steps {
                        sh '''
                            echo "Hello"
                            . venv/bin/activate
                            pip-audit > pip-audit-report.txt
                        '''
        
                        script {
                            def auditReport = readFile('pip-audit-report.txt')
                            if (auditReport.contains('Found')) {
                                error 'Found vulnerabilities in dependencies!'
                            }
                        }
                    }
                }
                
                stage('Python Safety Check'){
                    steps {
                       sh '''
                           . venv/bin/activate
                           safety scan --output html > safety-report.html
                       '''
                    }
                }
            }
        }

    }
    post {
            always {
                archiveArtifacts allowEmptyArchive: true, artifacts: 'pip-audit-report.txt', followSymlinks: false
                archiveArtifacts allowEmptyArchive: true, artifacts: 'safety-report.html', followSymlinks: false
            }
        }
}
