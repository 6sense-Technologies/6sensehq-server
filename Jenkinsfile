pipeline {
  agent { label 'docker-agent' }

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    GHCR_REPO = '6sensehq-server'
    SHORT_SHA = "${env.GIT_COMMIT.take(7)}"
  }

  stages {
    stage('📦 Checkout Source Code') {
      when {
        anyOf {
          branch 'main'
          branch 'master'
          branch 'test'
        }
      }
      steps {
        checkout scm
      }
    }

    stage('🚀 Deploy to Server') {
      when {
        anyOf {
          branch 'main'
          branch 'master'
          branch 'test'
        }
      }
      steps {
        script {
          def deployDir = "${env.GHCR_REPO}"
          withCredentials([usernamePassword(credentialsId: 'github-pat-6sensehq', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_PAT')]) {
            sshagent(credentials: ['ssh-6sensehq']) {
              sh """
                ssh -o StrictHostKeyChecking=no jenkins-deploy@95.216.144.222 "mkdir -p ~/${deployDir}"

                ssh -o StrictHostKeyChecking=no jenkins-deploy@95.216.144.222 '
                  set -e
                  cd ~/${deployDir}
                  echo "$GITHUB_PAT" | docker login ghcr.io -u $GITHUB_USER --password-stdin &&
                  if [ ! -d ".git" ]; then
                    echo "$GITHUB_PAT" | git clone -b ${env.BRANCH_NAME} ${env.GIT_URL} .  --password-stdin
                  else
                    echo "$GITHUB_PAT" | git fetch origin ${env.BRANCH_NAME} --password-stdin
                    echo "$GITHUB_PAT" | git reset --hard origin/${env.BRANCH_NAME} --password-stdin
                  fi

                  docker compose up -d --remove-orphans
                '
              """
            }
          }

          
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful"
    }
    failure {
      echo "❌ Deployment failed"
    }
  }
}