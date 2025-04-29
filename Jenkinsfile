pipeline {
    agent any
    stages {
        stage('Installing Dependencies') {
            steps {
                sh '''
                    echo "$PATH"
                    export PATH="$HOME/.local/bin:$PATH"
                    echo "$PATH"
                    pip install --upgrade pip
                    pip install --user pipenv
                    pipenv install 
                '''
            }
        }
    }
}
