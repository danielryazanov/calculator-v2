pipeline {
    agent none

    environment {
        APP_NAME = "calculator"
        APP_PORT = "5000"
        PROD_HOST = "<PROD_PUBLIC_IP>"   // תעדכן
    }

    stages {

        stage('Checkout') {
            agent { docker { image 'alpine/git' } }
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
                  pip install --no-cache-dir -r requirements.txt
                  pytest -q
                '''
            }
        }

        stage('Deploy to Production (CD)') {
            when {
                branch 'main'
            }
            agent {
                docker {
                    image 'alpine'
                }
            }
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'prod-ssh',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                      apk add --no-cache openssh-client
                      ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" $SSH_USER@''' + PROD_HOST + ''' '
                        docker rm -f calculator || true
                        docker run -d --name calculator \
                          -p 5000:5000 \
                          python:3.11-slim \
                          python api.py
                      '
                    '''
                }
            }
        }
    }
}
