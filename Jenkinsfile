pipeline {
    agent any
    stages {
        stage('Installing Dependencies') {
            steps {
                
                sh '''
                    export PATH="$HOME/.local/bin:$PATH"
                    pip install --upgrade pip
                    pip install --user pipenv
                    pipenv install 
                '''
            }
        }
    }
}
