// Send message to slack
def sendSlackNotification(String message, color='good') {
  slackSend(
      channel: '#cicd-notification',
      teamDomain: "${SLACK_WORKSPACE}",
      tokenCredentialId: 'SLACK_TOKEN',
      message: message,
      color: color
    )
}

def setupBranch{
  def envFile
  if (env.BRANCH_NAME == 'development') {
    envFile = env.DEVELOPMENT_ENV
    SERVER = env.DE
  } else if (env.BRANCH_NAME == 'staging') {
    envFile = env.STAGING_ENV
  } else if (env.BRANCH_NAME == 'main'){
    envFile = env.PRODUCTION_ENV
  }
}

pipeline {

  agent { node { label "lajoe" } }

  tools { nodejs "node:18.17.1" }

  environment {
    ENV_FILE = getEnv()
    SLACK_WORKSPACE = credentials('SLACK_WORKSPACE')
    SONARQUBE_LINK_GLOBAL = credentials('SONARQUBE_LINK_GLOBAL')
    REPOSITORY_NAME = sh(returnStdout: true, script: 'echo ${JOB_NAME} | cut -d "/" -f1').trim()
    GITLOG = sh(returnStdout: true, script: 'git log --format="Author: %an | Commit ID: %h\n Commit Message: %s" -1')
    HOME = '.'
  }

  options {
    disableConcurrentBuilds()
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(daysToKeepStr: "7",artifactDaysToKeepStr: "7"))
  }

  stages {

    stage('Notify Slack Channel') {
      steps {
        sendSlackNotification(
        "🛬 Starting CI/CD for ${env.JOB_NAME}\n" +
        "Build Number: ${env.BUILD_NUMBER} | ${GITLOG}" +
        "<${env.BUILD_URL}console|Console Output> || <${env.JOB_URL}|Jobs Dashboard> || <${env.JOB_DISPLAY_URL}/${env.BRANCH_NAME}|Blue Ocean Dashboard> || <${SONARQUBE_LINK_GLOBAL}${REPOSITORY_NAME}%3A${env.BRANCH_NAME}|Sonarqube>")
      }
    }

    stage('Performing SonarQube Analysis') {
      environment {
        SCANNERHOME = tool 'SONARSCANNER'
      }
      steps {
        withSonarQubeEnv('SONARQUBE_SERVER') {
          sh """
          ${SCANNERHOME}/sonar-scanner-4.8.0.2856-linux/bin/sonar-scanner \
          -D sonar.projectKey=${REPOSITORY_NAME}:${env.BRANCH_NAME} \
          -D sonar.projectName=${REPOSITORY_NAME}:${env.BRANCH_NAME} \
          """
        }
      }
    }

    stage('Build') {
      steps {
        script {
          def ENV_FILE

          if (env.BRANCH_NAME == 'development') {
            ENV_FILE = env.DEVELOPMENT_ENV
          } else if (env.BRANCH_NAME == 'staging') {
            ENV_FILE = env.STAGING_ENV
          } else if (env.BRANCH_NAME == 'main'){
            ENV_FILE = env.PRODUCTION_ENV
          }
        }

        withCredentials([file(credentialsId: ENV_FILE, variable: 'ENV_FILE')]) {
          sh 'mv ${ENV_FILE} .env'
          sh 'rm -rf package-lock.json'
          sh 'npm install'
          sh 'npm run build'
        }
      }
    }

    stage('Development: Send Artifact to Server') {
      when { branch "development" }
      steps {
        withCredentials([string(credentialsId: "DEVELOPMENT_SERVER", variable: 'DEVELOPMENT_SERVER')]) {
          sshagent(credentials: ['PRIVATE_KEY']){
            sh 'rsync -avz --delete build/ ${DEVELOPMENT_SERVER}:~/fe-dumbmerch/'
          }
        }
      }
    }

    stage('staging: Send Artifact to Server') {
      when { branch "staging" }
      steps {
        withCredentials([string(credentialsId: "DEVELOPMENT_SERVER", variable: 'DEVELOPMENT_SERVER')]) {
          sshagent(credentials: ['PRIVATE_KEY']){
            sh 'rsync -avz --delete build/ ${DEVELOPMENT_SERVER}:~/fe-dumbmerch/'
          }
        }
      }
    }

  }

  post {
    failure {
      sendSlackNotification('The build process has failed. Please review the build log and contact the administrator for assistance.', 'danger')
    }
    aborted {
      sendSlackNotification('The build process has been manually aborted.', 'warning')
    }
    success {
      sendSlackNotification('The build process has completed successfully. ', 'good')
    }
  }
}