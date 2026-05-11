pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'iremgeciit' // Burayı kendi Docker Hub kullanıcı adınla değiştir!
        SONAR_HOST      = 'http://techstore-sonarqube:9000' // localhost yerine konteyner adı
        SONAR_TOKEN     = credentials('sonar-token')
        SLACK_CHANNEL   = '#devops-techstore'
    }

    stages {
        // ── 1. KAYNAK KOD ───────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Kod GitHub'dan alındı"
            }
        }

        // ── 2. ORTAM KURULUMU ───────────────────────────────────
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

        // ── 3. BİRİM TESTLERİ ──────────────────────────────────
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

        // ── 4. KOD KALİTE ANALİZİ ──────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        . venv/bin/activate
                        sonar-scanner \
                            -Dsonar.projectKey=techstore \
                            -Dsonar.projectName="TechStore E-Commerce" \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.host.url=${SONAR_HOST} \
                            -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }

        // ── 5. KALİTE KAPISI ───────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                echo "✅ SonarQube kalite kapısı geçildi"
            }
        }

        // ── 6. DOCKER İMAJI ─────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:latest ."
                echo "✅ Docker imajı oluşturuldu"
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline başarıyla tamamlandı!"
        }
        failure {
            echo "❌ Pipeline başarısız!"
        }
        always {
            cleanWs()
        }
    }
}

