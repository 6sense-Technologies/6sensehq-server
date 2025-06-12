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
      
    }
    failure {
      
    }
  }
}

// -------------------
// GitHub API Helpers
// -------------------
def getRepoFromGitUrl() {
  def url = env.GIT_URL
  if (!url || url.trim() == '') {
    url = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
  }
  if (url.startsWith("git@github.com:")) {
    return url.replace("git@github.com:", "").replace(".git", "")
  } else if (url.startsWith("https://github.com/")) {
    return url.replace("https://github.com/", "").replace(".git", "")
  } else {
    error("Unknown Git URL format: ${url}")
  }
}

def createAndUpdateGitHubDeployment(String repo, String ref, String branch, String deployEnv, String deployUrl) {
  withCredentials([usernamePassword(credentialsId: 'github-pat-6sensehq', usernameVariable: 'GH_USER', passwordVariable: 'GITHUB_PAT')]) {
    withEnv(["TOKEN=${GITHUB_PAT}"]) {
      def description = "Deployed from Jenkins pipeline for branch ${branch}"
      def rawResponse = sh(
        script: """
          curl -s -X POST \\
            -H 'Authorization: token $TOKEN' \\
            -H 'Accept: application/vnd.github+json' \\
            https://api.github.com/repos/${repo}/deployments \\
            -d '{
              "ref": "${ref}",
              "task": "deploy",
              "auto_merge": false,
              "required_contexts": [],
              "description": "${description}",
              "environment": "${deployEnv}",
              "environment_url": "${deployUrl}",
              "sha": "${ref}"
            }'
        """,
        returnStdout: true
      ).trim()

      def deploymentId = sh(
        script: """
          echo '${rawResponse}' | grep -Eo '"id":[ ]*[0-9]+' | head -n1 | cut -d':' -f2
        """,
        returnStdout: true
      ).trim()

      if (!deploymentId || deploymentId == "null") {
        error("❌ Failed to extract deployment ID")
      }

      return deploymentId
    }
  }
}


def updateGitHubDeploymentStatus(String repo, String logUrl, String deploymentId, String status, String deployEnv, String deployUrl) {
  withCredentials([usernamePassword(credentialsId: 'github-pat-6sensehq', usernameVariable: 'GH_USER', passwordVariable: 'GITHUB_PAT')]) {
    withEnv(["TOKEN=${GITHUB_PAT}"]) {
      sh """
        curl -s -X POST \\
          -H 'Authorization: token $TOKEN' \\
          -H "Accept: application/vnd.github+json" \\
          https://api.github.com/repos/${repo}/deployments/${deploymentId}/statuses \\
          -d '{
            "state": "${status}",
            "log_url": "${logUrl}",
            "description": "Deployment ${status}",
            "environment": "${deployEnv}",
            "environment_url": "${deployUrl}"
          }'
      """
    }
    
  }
}