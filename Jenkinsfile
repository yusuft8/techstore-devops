pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'yusufft8'
        SONAR_HOST      = 'http://sonarqube:9000'
	SONAR_TOKEN     = credentials('sonarqube-token')
        SLACK_CHANNEL   = '#devops-techstore'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Kod GitHub'dan alindi: ${env.GIT_COMMIT?.take(7)}"
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
                echo "Python sanal ortami hazir"
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    mkdir -p test-results
                    cd $WORKSPACE
		    export PYTHONPATH=$WORKSPACE
                    pytest tests/test_app.py \
                        -v \
                        --tb=short \
                        --junit-xml=test-results/unit-tests.xml \
                        --cov=app \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results/unit-tests.xml'
                }
            }
        }
     stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                            -Dsonar.projectKey=techstore \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.host.url=http://techstore-sonarqube:9000 \
                            -Dsonar.token=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                        -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                """
                echo "Docker imaji olusturuldu: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                        docker tag ${DOCKER_IMAGE}:latest \$DOCKER_USER/${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                        docker tag ${DOCKER_IMAGE}:latest \$DOCKER_USER/${DOCKER_IMAGE}:latest
                        docker push \$DOCKER_USER/${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                        docker push \$DOCKER_USER/${DOCKER_IMAGE}:latest
                    """
                }
                echo "Imaj Docker Hub'a yuklendi"
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker stop techstore-app 2>/dev/null || true
                    docker rm techstore-app 2>/dev/null || true
                    docker run -d \
                        --name techstore-app \
                        --restart unless-stopped \
                        -p 5000:5000 \
                        ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest
                    sleep 10
                """
            }
        }

        
    }

    post {
        success {
            echo "Pipeline basariyla tamamlandi!"
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: "TechStore Deploy Basarili - Build #${env.BUILD_NUMBER}"
            )
        }
        failure {
            echo "Pipeline basarisiz!"
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: "TechStore Deploy Basarisiz - Build #${env.BUILD_NUMBER} - ${env.BUILD_URL}console"
            )
        }
        always {
            sh "docker image prune -f --filter 'until=72h' || true"
            cleanWs()
        }
    }
}
