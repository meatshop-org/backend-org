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
                '''
            }
        }

        stage('Audit Dependencies') {
            steps {
                sh '''
                    python3.11 -m venv venv
                    . venv/bin/activate
                    pip-audit
                '''
            }
        }
    }
}
