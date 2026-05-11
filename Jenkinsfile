pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'techstore-app'
        DOCKER_HUB_USER = 'iremgecit'
        SONAR_HOST = 'http://techstore-sonarqube:9000'
        SONAR_TOKEN = credentials('sonar-token')
        SLACK_CHANNEL = '#devops-techstore'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Kod GitHub'dan alındı: ${env.GIT_COMMIT}"
            }
        }

        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
                echo "✅ Python sanal ortamı hazır"
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    PYTHONPATH=. pytest tests/test_app.py -v --tb=short --junit-xml=test-results/unit-tests.xml --cov=app --cov-report=xml:coverage.xml --cov-report=term-missing
                '''
            }
            post {
                always {
                    junit 'test-results/unit-tests.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            . venv/bin/activate
                            /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarScanner/bin/sonar-scanner \
                            -Dsonar.projectKey=techstore \
                            -Dsonar.projectName='TechStore E-Commerce' \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                            -Dsonar.python.coverage.reportPaths=coverage.xml
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest ."
                echo "✅ Docker imajı oluşturuldu"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
                    '''
                }
                echo "✅ Docker Hub'a gönderildi"
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker stop techstore-app || true
                    docker rm techstore-app || true
                    docker run -d --name techstore-app -p 5000:5000 ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
                '''
                echo "✅ Uygulama dağıtıldı"
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    sleep 15
                    curl -f http://localhost:5000/health || exit 1
                '''
                echo "✅ Smoke test geçti"
            }
        }

        stage('UI Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    PYTHONPATH=. pytest tests/test_ui.py -v --tb=short
                '''
                echo "✅ UI testleri geçti"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ Pipeline başarılı! TechStore çalışıyor: http://localhost:5000"
            slackSend(
                channel: SLACK_CHANNEL,
                color: 'good',
                message: "✅ Pipeline başarılı! Uygulama dağıtıldı."
            )
        }
        failure {
            echo "❌ Pipeline başarısız!"
            slackSend(
                channel: SLACK_CHANNEL,
                color: 'danger',
                message: "❌ Pipeline başarısız! Lütfen logları kontrol edin."
            )
        }
    }
}
