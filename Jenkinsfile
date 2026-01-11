pipeline {
  agent none

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    AWS_REGION = "us-east-1"

    // ECR (נתת)
    ECR_REPO = "992382545251.dkr.ecr.us-east-1.amazonaws.com/calculator-app-daniel"

    APP_NAME  = "calculator"
    APP_PORT  = "5000"

    // PROD (לפי התמונה שלך)
    PROD_HOST = "3.91.231.32"
    PROD_USER = "ec2-user"

    // tags
    IMAGE_TAG    = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    IMAGE_FULL   = "${env.ECR_REPO}:${env.IMAGE_TAG}"
    IMAGE_LATEST = "${env.ECR_REPO}:latest"
  }

  stages {

    stage('Checkout') {
      agent any
      steps {
        checkout scm
      }
    }

    stage('Build & Test (CI)') {
      agent {
        docker {
          image 'python:3.11-slim'
          args '-u root:root'
        }
      }
      steps {
        sh '''
          set -e

          python --version
          pip install --upgrade pip

          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          fi

          pip install pytest

          # FIX: allow tests to import modules from repo root (calculator_app, calculator_logic)
          export PYTHONPATH="$PWD:$PYTHONPATH"

          pytest -v
        '''
      }
    }

    stage('Build Docker Image') {
      agent {
        docker {
          image 'docker:27-cli'
          args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root'
        }
      }
      steps {
        sh '''
          set -e
          docker version
          docker build -t "$IMAGE_FULL" -t "$IMAGE_LATEST" .
        '''
      }
    }

    stage('Push Image (ECR)') {
      agent {
        docker {
          image 'docker:27-cli'
          args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root'
        }
      }
      steps {
        sh '''
          set -e

          # install aws cli inside the docker agent
          apk add --no-cache aws-cli >/dev/null

          aws --version

          # Login to ECR using Instance Role (preferred).
          # If you don't have Instance Role, create Jenkins credential 'aws-access' (username/password)
          # and Jenkins will inject AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY automatically via withCredentials below.
        '''

        // אם יש לך Instance Role על Jenkins EC2 (מומלץ) – זה עדיין יעבוד גם בלי aws-access.
        // אם אין Role – צור Credentials בשם aws-access (Username=AKIA..., Password=SECRET).
        withCredentials([usernamePassword(credentialsId: 'aws-access', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -e
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "992382545251.dkr.ecr.us-east-1.amazonaws.com"

            docker push "$IMAGE_FULL"
            docker push "$IMAGE_LATEST"
          '''
        }
      }
    }

    stage('Deploy to Production (CD)') {
      when { branch 'main' }

      agent {
        docker {
          image 'alpine:3.20'
          args '-u root:root'
        }
      }

      steps {
        // Credentials: SSH Username with private key
        // credentialsId must be: prod-ssh-key
        // Username inside credentials can be anything; we use PROD_USER env for actual login user.
        withCredentials([sshUserPrivateKey(credentialsId: 'prod-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh '''
            set -e
            apk add --no-cache openssh-client bash >/dev/null

            chmod 600 "$SSH_KEY"

            echo "Deploying $IMAGE_FULL to $PROD_USER@$PROD_HOST ..."

            ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" "$PROD_USER@$PROD_HOST" /bin/bash -s <<'EOSSH'
              set -e

              AWS_REGION="'"$AWS_REGION"'"
              ECR_REPO="'"$ECR_REPO"'"
              IMAGE="'"$IMAGE_FULL"'"
              APP_NAME="'"$APP_NAME"'"
              APP_PORT="'"$APP_PORT"'"

              # Ensure docker
              docker --version

              # Ensure aws-cli exists (Amazon Linux 2)
              if ! command -v aws >/dev/null 2>&1; then
                sudo yum install -y awscli
              fi
              aws --version

              # Login to ECR (PROD instance should have Instance Role that allows ECR pull)
              aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "${ECR_REPO%/*}"

              docker pull "$IMAGE"

              # Replace container
              if docker ps -a --format '{{.Names}}' | grep -q "^${APP_NAME}$"; then
                docker rm -f "$APP_NAME" || true
              fi

              docker run -d --name "$APP_NAME" --restart unless-stopped -p "${APP_PORT}:${APP_PORT}" "$IMAGE"

              docker ps --filter "name=$APP_NAME"
EOSSH
          '''
        }
      }
    }

    stage('Health Verification') {
      when { branch 'main' }
      agent {
        docker {
          image 'curlimages/curl:8.10.1'
        }
      }
      steps {
        sh '''
          set -e
          echo "Health check: http://$PROD_HOST:$APP_PORT/"

          for i in 1 2 3 4 5 6 7 8 9 10; do
            if curl -fsS "http://$PROD_HOST:$APP_PORT/" >/dev/null; then
              echo "OK"
              exit 0
            fi
            echo "Not ready yet... ($i/10)"
            sleep 3
          done

          echo "Health check failed"
          exit 1
        '''
      }
    }
  }

  post {
    always {
      echo "Build finished: ${currentBuild.currentResult}"
    }
  }
}

