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
        RUNNING_BACKEND = 'http://ec2-157-175-219-194.me-south-1.compute.amazonaws.com/'
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
                    python3.11 -m pip install coverage
                    python3.11 -m pip install drf-spectacular
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
        stage('Integration Testing - AWS EC2') {
            when {
                branch "feature/*"
            }
            steps {
                withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'me-south-1') {
                    sh '''
                        bash integration-testing-ec2.sh
                    '''
                }
            }
        }
        stage('Deploy - AWS EC2') {
            when {
                branch 'feature/*'
            }
            steps {
                script {
                    sshagent(['aws-dev-deploy-ec2-instance']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@157.175.219.194 '
                                sudo docker image prune -a -f
                                sudo docker network create meatshop-net
                                sudo docker rm -f \$(sudo docker ps -q)
                                if docker ps -a | grep -q "mymysql"; then
                                    echo "Container Found, Stopping..."
                                    docker stop "mymysql" && docker rm "mymysql"
                                    echo "Container stopped and removed"
                                fi
                                docker run -d --name mymysql --network meatshop-net -e MYSQL_ROOT_PASSWORD=mypass -e MYSQL_DATABASE=meatshop -p 3306:3306 -v mysql_data:/var/lib/mysql mysql

                                if sudo docker ps -a | grep -q "backend"; then
                                    echo "Container Found, Stopping..."
                                    sudo docker stop "backend" && sudo docker rm "backend"
                                    echo "Container stopped and removed"
                                fi
                                sudo docker run -d \
                                    --network meatshop-net \
                                    -e DB_NAME=${DB_NAME} \
                                    -e DB_PORT=${DB_PORT} \
                                    -e LOCAL_DB_HOST=mymysql \
                                    -e LOCAL_DB_USER=${LOCAL_DB_USER} \
                                    -e LOCAL_DB_PASSWORD=${LOCAL_DB_PASSWORD} \
                                    -p 80:8000 --name backend borhom11/meatshop-backend:$GIT_COMMIT
                            '
                        """
                    }
                }
            }   
        }
         stage('K8S Update Image Tag') {
            when {
                branch 'PR*'
            }
            steps {
                sh 'git clone -b main https://github.com/BRHM1/k8s-meatshop.git'
                dir('k8s-meatshop/backend') {
                    sh '''
                        git checkout main
                        git checkout -b feature-$BUILD_ID
                        sed -E -i "s~(eladwy|borhom11)/[^ ]*~borhom11/meatshop-backend:$GIT_COMMIT~g" deployment.yaml

                        git config --global user.email $USER_EMAIL
                        git remote set-url origin https://$GITHUB_TOKEN@github.com/BRHM1/k8s-meatshop.git
                        git add . 
                        git commit -m "FROM CI/CD - Update image tag to $GIT_COMMIT"
                        git push origin feature-$BUILD_ID
                    '''
                }
            }
        }
        stage('K8S Raise PR Review') {
            when {
                branch 'PR*'
            }
            steps {
                sh '''
                    curl -L \
                        -X POST \
                        -H "Accept: application/vnd.github+json" \
                        -H "Authorization: Bearer $FGGITHUB_TOKEN" \
                        -H "X-GitHub-Api-Version: 2022-11-28" \
                        https://api.github.com/repos/BRHM1/k8s-meatshop/pulls \
                        -d '{"title":"Raised PR From CI/CD","body":"Please pull these awesome changes in!","head":"feature-'"$BUILD_ID"'","base":"main"}'
                '''
            }
        }
        stage('PR merged & ArgoCD synced?') {
            when {
                branch 'PR*'
            }
            steps{
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Confirm that the manifest repo PR is merged and ArgoCD is synced.', ok: 'YES! All Done', submitter: 'admin'
                }
            }
        }
        stage('DAST - OWASP ZAP') {
            when {
                branch 'PR*'
            }
            steps {
                sh '''
                    echo Trigger
                    chmod 777 $(pwd)
                    docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-api-scan.py \
                        -t http://ec2-157-175-219-194.me-south-1.compute.amazonaws.com/schema/?format=json \
                        -r zap_report.html \
                        -f openapi \
                        -w zap_report.md \
                        -x zap_report.xml \
                        -J zap_report.json \
                        -c zap_ignore_rules
                '''
            }
        }

        stage('Publish Reports - AWS S3') {
            when {
                branch 'PR*'
            }
            steps {
                withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'me-south-1') {
                    sh '''
                        mkdir reports-$BUILD_ID
                        echo reports
                        cp coverage.xml reports-$BUILD_ID/
                        cp trivy*.* reports-$BUILD_ID/
                        cp pip-* reports-$BUILD_ID/
                        ls reports-$BUILD_ID/
                    '''
                    s3Upload(
                        file: "reports-$BUILD_ID",
                        bucket: "meatshop-pipeline-reports",
                        path: "backend/reports-$BUILD_ID"
                    )
                }
            }
        }

    }
    post {
            always {
                script {
                    if (fileExists('k8s-meatshop')) {
                        sh 'rm -rf k8s-meatshop'
                    }
                }
                junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-MEDIUM-results.xml'
                junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-CRITICAL-results.xml'

                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image Medium vulnarability Report', reportTitles: '', useWrapperFileDirectly: true])
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image CRITICAL vulnarability Report', reportTitles: '', useWrapperFileDirectly: true])
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'zap_report.html', reportName: 'DAST - OWASP ZAP Report', reportTitles: '', useWrapperFileDirectly: true])
                archiveArtifacts allowEmptyArchive: true, artifacts: 'pip-audit-report.txt', followSymlinks: false
            }
        }
}
