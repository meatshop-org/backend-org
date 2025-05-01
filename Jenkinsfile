pipeline {
    agent any
    environment {
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner-710'
        GITHUB_TOKEN = credentials('github-pat')
        USER_EMAIL = credentials('github-email')
        FGGITHUB_TOKEN = credentials('FGgithub-pat')
        DB_NAME = 'meatshop'
        LOCAL_DB_HOST = 'localhost'
        LOCAL_DB_USER = 'root'
        LOCAL_DB_PASSWORD = 'mypass'
        DB_PORT = '3306'
        EC2_URL = ''
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
                    python3.11 -m pip install coverage
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
        stage('Run DB') {
            steps {
                script {
                    sh '''
                        echo Hello
                        if docker ps -a | grep -q "mymysql"; then
                            echo "Container Found, Stopping..."
                            docker stop "mymysql" && docker rm "mymysql"
                            echo "Container stopped and removed"
                        fi
                        docker run -d --name mymysql --network meatshop-net -e MYSQL_ROOT_PASSWORD=mypass -e MYSQL_DATABASE=meatshop -p 3306:3306 -v mysql_data:/var/lib/mysql mysql
                    '''
                }
            }
        }
        stage('Run Unit Tests') {
            steps {
                 sh ''' 
		            sleep 60
                    . venv/bin/activate
                    python3.11 manage.py test --no-input --failfast
                '''
            }
        }
        stage('Code Coverage') {
            steps {
                 sh ''' 
		            sleep 60
                    . venv/bin/activate
                    coverage run --source='.' manage.py test --no-input --failfast
		            coverage xml -o coverage.xml
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
                              -Dsonar.sources=tags/,shop/,meatshop/,likes/,core/ \
			                  -Dsonar.python.coverage.reportPaths=coverage.xml
                         '''
                    }
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t borhom11/meatshop-backend:$GIT_COMMIT .'
                // script {
                //     sh '''
                //         if docker ps -a | grep -q "backend"; then
                //             echo "Container Found, Stopping..."
                //             docker stop "backend" && docker rm "backend"
                //             echo "Container stopped and removed"
                //         fi
                //         docker run -d \
                //             --network meatshop-net \
                //             -e DB_NAME=${DB_NAME} \
                //             -e DB_PORT=${DB_PORT} \
                //             -e LOCAL_DB_HOST=mymysql \
                //             -e LOCAL_DB_USER=${LOCAL_DB_USER} \
                //             -e LOCAL_DB_PASSWORD=${LOCAL_DB_PASSWORD} \
                //             -p 8089:8000 --name backend borhom11/meatshop-backend:$GIT_COMMIT
                //     '''
                // }
            }
        }
        stage('Trivy Vulnarability Scanner'){
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        trivy image borhom11/meatshop-backend:$GIT_COMMIT \
                            --severity LOW,MEDIUM \
                            --exit-code 0 \
                            --quiet \
                            --format json -o trivy-image-MEDIUM-results.json
        
                        trivy image borhom11/meatshop-backend:$GIT_COMMIT \
                            --severity HIGH,CRITICAL \
                            --exit-code 1 \
                            --quiet \
                            --format json -o trivy-image-CRITICAL-results.json
                    '''
                }
            }
            post {
                always {
                    sh '''
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-MEDIUM-results.html trivy-image-MEDIUM-results.json
                            
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-CRITICAL-results.html trivy-image-CRITICAL-results.json
                            
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-MEDIUM-results.xml trivy-image-MEDIUM-results.json
                            
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json
                    '''
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                withDockerRegistry(url: 'https://index.docker.io/v1/', credentialsId: 'docker-hub-creds') {
                    sh 'docker push borhom11/meatshop-backend:$GIT_COMMIT'
                }
            }   
        }

        stage('Integration Testing - GET AWS EC2 URL') {
            when {
                branch "feature/*"
            }
            steps {
                withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'me-south-1') {
                    script {
                        def url = sh(script: 'bash integration-testing-ec2.sh', returnStdout: true).trim()
                        env.EC2_URL = url
                        echo "EC2 Instance URL: ${env.EC2_URL}"
                    }
                }
            }
        }

        stage('Testing URL') {
            steps {
                sh 'echo $EC2_URL'
                sh 'echo Hello'
            }
        }

    }
    post {
            always {
                junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-MEDIUM-results.xml'
                junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-CRITICAL-results.xml'

                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image Medium vulnarability Report', reportTitles: '', useWrapperFileDirectly: true])
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image CRITICAL vulnarability Report', reportTitles: '', useWrapperFileDirectly: true])

                archiveArtifacts allowEmptyArchive: true, artifacts: 'pip-audit-report.txt', followSymlinks: false
            }
        }
}
