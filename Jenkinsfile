pipeline {
    agent any

    environment {
        IMAGE_NAME = 'codewithchandresh10000/flask-jenkins-demo'
        CONTAINER_NAME = 'flask-devsecops-app'
        SONAR_SCANNER_HOME = tool 'SonarScanner'
    }

    stages {

        // 1. CHECKOUT
        stage('Checkout') {
            steps {
                git branch:'main' url: 'https://github.com/codewithchandresh10000/devsecops.git'
            }
        }

        // 2. INSTALL DEPENDENCIES
        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    .venv/bin/python -m pip install --upgrade pip
                    .venv/bin/python -m pip install -r requirements.txt
                    .venv/bin/python -m pip install bandit pytest
                '''
            }
        }

        // 3. PYTHON TESTING
        stage('Run Tests') {
            steps {
                sh '.venv/bin/python -m pytest -v'
            }
        }

        // 4. BANDIT SECURITY SCAN
        stage('Bandit Security Scan') {
            steps {
                sh '''
                    .venv/bin/bandit -r . \
                    -x ./.venv,./tests \
                    -ll
                '''
            }
        }

        // 5. SONARQUBE ANALYSIS
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=flask-jenkins-demo \
                        -Dsonar.projectName=flask-jenkins-demo \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=.venv/,tests/ \
                        -Dsonar.python.version=3.12
                    '''
                }
            }
        }

        // 6. SONARQUBE QUALITY GATE
        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // 7. BUILD DOCKER IMAGE
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        // 8. TRIVY SECURITY SCAN
        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --ignore-unfixed \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        // 9. TEST DOCKER CONTAINER
        stage('Test Docker Container') {
            steps {
                sh '''
                    docker rm -f flask-test || true
                    
                    docker run -d \
                      --name flask-test \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    sleep 3

                    docker exec flask-test python -c \
                      "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:5000/health').read().decode())"

                    docker rm -f flask-test || true
                '''
            }

            post {
                always {
                    sh 'docker rm -f flask-test || true'
                }
            }
        }

        // 10. PUSH TO DOCKER HUB
        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Dockerhub',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKERHUB_TOKEN" | \
                          docker login -u "$DOCKERHUB_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest

                        docker logout
                    '''
                }
            }
        }

        // 11. DEPLOY CONTAINER
        stage('Deploy Container') {
            steps {
                sh '''
                    docker rm -f ${CONTAINER_NAME} || true

                    docker run -d \
                      --restart unless-stopped \
                      --name ${CONTAINER_NAME} \
                      -p 5000:5000 \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    sleep 3

                    docker exec ${CONTAINER_NAME} python -c \
                      "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:5000/health').read().decode())"
                '''
            }
        }
    }

    // POST ACTIONS
    post {
        success {
            echo 'SUCCESS: Flask application built, tested, scanned, pushed and deployed!'
        }

        failure {
            echo 'FAILED: Check Jenkins Console Output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
