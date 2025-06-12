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

          sshagent(credentials: ['ssh-6sensehq']) {
            sh """
              ssh -o StrictHostKeyChecking=no jenkins-deploy@95.216.144.222 "mkdir -p ~/${deployDir}"
              scp -o StrictHostKeyChecking=no .[^.]* * jenkins-deploy@95.216.144.222:~/${deployDir}/
              
              ssh -o StrictHostKeyChecking=no jenkins-deploy@95.216.144.222 '
                cd ~/${deployDir} &&
                docker compose up -d --remove-orphans
              '
            """
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