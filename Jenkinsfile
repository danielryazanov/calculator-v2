pipeline {
    agent none

    environment {
        APP_NAME  = "calculator"
        APP_PORT  = "5000"
        // PROD_HOST = "<PROD_PUBLIC_IP>"  // תגדיר בהמשך
    }

    stages {

        stage('Checkout') {
            agent {
                docker {
                    image 'python:3.11-slim'
                }
            }
            steps {
                checkout scm
            }
        }

        stage('Build & Test (CI)') {
            agent {
                docker {
                    image 'python:3.11-slim'
                }
            }
            steps {
                sh '''
                  python --version
                  pip install --upgrade pip
                  if [ -f requirements.txt ]; then
                    pip install -r requirements.txt
                  fi
                  pytest -v
                '''
            }
        }

        stage('Build & Push Image (ECR)') {
            when {
                branch 'main'
            }
            agent {
                docker {
                    image 'docker:27-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                  docker version
                  echo "Docker build & push stage"
                  # כאן בהמשך נכניס ECR login + build + push
                '''
            }
        }

        stage('Deploy to Production (CD)') {
            when {
                branch 'main'
            }
            agent {
                docker {
                    image 'docker:27-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                  echo "Deploy stage placeholder"
                  # כאן בהמשך יהיה ssh + docker run על שרת ה־prod
                '''
            }
        }
    }
}
