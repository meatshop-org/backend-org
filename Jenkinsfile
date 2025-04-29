pipeline {
    agent any
    stages {
        stage('Installing Dependencies') {
            steps {
                sh '''
                    pipenv --version
                    pip --version
                    python3 --version
                    pipenv install 
                '''
            }
        }
    }
}
