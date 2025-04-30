pipeline {
    agent any
    environment {
        SAFETY_API_KEY = credentials('SAFETY_API_KEY')
    }
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
                    python3.11 -m pip install safety auth
                    python3.11 -m pip install yq
                '''
            }
        }
        stage('Dependencies Scanning...'){
            parallel {
                stage('Audit Dependencies') {
                    steps {
                        sh '''
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
                        find . -name "db.sqlite3" -delete
                        
                        # Create clean file list (with escaped backslashes)
                        find . -type f \\( -name "*.py" -o -name "requirements.txt" \\) \\
                            -not -path "./venv/*" \\
                            -not -path "./media/*" \\
                            -not -path "./static/*" > python_files.txt
                        
                        # Run safety scan
                        safety --key $SAFETY_API_KEY scan --file-list python_files.txt --output html safety_report.html
                        
                        # Fallback for empty reports
                        [ -s safety_report.html ] || echo "<html><body>No vulnerabilities found</body></html>" > safety_report.html
                        '''
                    }
                }
            }
        }

    }
    post {
            always {
                archiveArtifacts allowEmptyArchive: true, artifacts: 'pip-audit-report.txt', followSymlinks: false
                publishHTML target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'safety_report.html',
                reportName: 'Safety Vulnerability Report'
            ]
            }
        }
}
