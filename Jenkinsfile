pipeline {
    agent any
    environment {
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner-710'
        GITHUB_TOKEN = credentials('github-pat')
        USER_EMAIL = credentials('github-email')
        FGGITHUB_TOKEN = credentials('FGgithub-pat')
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
        stage('Audit Dependencies') {
            steps {
                 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        . venv/bin/activate
                        pip-audit > pip-audit-report.txt
                    '''
                }
            }
        }
        stage('Run Unit Tests') {
            steps {
                 sh ''' 
                     python manage.py test --no-input --parallel --failfast
                '''
            }
        }
        stage('SAST - SonarQube') {
            steps {
                timeout(time: 540, unit: 'SECONDS'){
                    withSonarQubeEnv('sonar-qube-backend-server') {
                        sh '''
                            $SONAR_SCANNER_HOME/bin/sonar-scanner \
                              -Dsonar.projectKey=backend-project \
                              -Dsonar.sources=. \
                         '''
                    }
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        waitForQualityGate abortPipeline: true
                    }
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
