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
                      sh """
                            . venv/bin/activate
                        
                            safety generate policy_file
                        
                            awk '/include-files: \\[\\]/ {
                                print "  include-files: ["requirements.txt", "Pipfile.lock"]";
                                next
                            }
                            { print }' .safety-policy.yml > tmp && mv tmp .safety-policy.yml
                        
                            safety --key \$SAFETY_API_KEY scan --policy .safety-policy.yml
                    """
                    }
                }
            }
        }

    }
    post {
            always {
                archiveArtifacts allowEmptyArchive: true, artifacts: 'pip-audit-report.txt', followSymlinks: false
                // archiveArtifacts allowEmptyArchive: true, artifacts: 'output.html', followSymlinks: false
            }
        }
}
