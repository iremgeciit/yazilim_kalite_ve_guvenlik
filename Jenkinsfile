pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'iremgeciit' 
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Kod GitHub'dan alındı"
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
                    PYTHONPATH=. pytest tests/test_app.py -v --tb=short --junitxml=test-results/unit-tests.xml --cov=app --cov-report=xml:coverage.xml --cov-report=term-missing
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
                    // Jenkins Tools'ta tanımladığın 'SonarScanner' ismini buraya bağladık
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            . venv/bin/activate
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=techstore \
                                -Dsonar.projectName="TechStore E-Commerce" \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                                -Dsonar.python.coverage.reportPaths=coverage.xml
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                echo "✅ SonarQube kalite kapısı geçildi"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:latest ."
                echo "✅ Docker imajı oluşturuldu"
            }
        }
    }

    post {
        success { echo "🎉 Pipeline başarıyla tamamlandı!" }
        failure { echo "❌ Pipeline başarısız!" }
        always { cleanWs() }
    }
}
         
          
