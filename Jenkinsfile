pipeline {
    agent any
    stages {
        stage('Installing Dependencies') {
            steps {
                sh '''
                    export PATH="$HOME/.local/bin:$PATH"
                    pipenv --version
                    pip --version
                    python3 --version
                    pipenv install 
                '''
            }
        }
    }
}
