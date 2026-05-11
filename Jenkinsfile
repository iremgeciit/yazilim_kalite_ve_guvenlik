pipeline {
    agent any

  environment {
    DOCKER_IMAGE    = 'techstore-app'
    DOCKER_HUB_USER = 'iremgecit' 
    SONAR_URL       = 'http://techstore-sonarqube:9000'
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
            echo "✅ Python ortamı hazır"
        }
    }

    stage('Unit Tests') {
        steps {
            sh '''
                . venv/bin/activate
                PYTHONPATH=. pytest tests/test_app.py -v --tb=short --junitxml=test-results/unit-tests.xml --cov=app --cov-report=xml:coverage.xml
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
                def scannerHome = tool 'SonarScanner'
                withSonarQubeEnv('SonarQube') {
                    sh """
                        . venv/bin/activate
                        ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=techstore \
                            -Dsonar.projectName="TechStore E-Commerce" \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.host.url=${env.SONAR_URL}
                    """
                }
            }
        }
    }

    stage('Quality Gate') {
        steps {
            waitForQualityGate abortPipeline: true
        }
    }

    stage('Build Docker Image') {
        steps {
            sh "docker build -t ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest ."
        }
    }

    stage('Push to Docker Hub') {
        steps {
            withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                sh """
                    echo \$PASS | docker login -u \$USER --password-stdin
                    docker push ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
                """
            }
        }
    }

    stage('Deploy') {
        steps {
            sh """
                docker stop techstore-app || true
                docker rm techstore-app || true
                docker run -d --name techstore-app -p 5000:5000 ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
            """
            echo "✅ Uygulama yayında"
        }
    }

    stage('Smoke Test') {
        steps {
            sh '''
                sleep 30
                curl -v http://techstore-app:5000/health || curl -v [http://172.17.0.1:5000/health](http://172.17.0.1:5000/health)
            '''
            echo "✅ Smoke Test Başarılı!"
        }
    }
}

post {
    success {
        echo "🎉 TEBRİKLER İREM! HER ŞEY YEŞİL!"
    }
    always {
        cleanWs()
    }
}

}
