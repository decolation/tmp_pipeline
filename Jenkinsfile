
pipeline {

  agent {
    node {
      label ''
    }
  }

  // global env variables
  environment {
//    NODEJS_HOME = tool 'nodejs11.13'
//    PATH='${PATH}:/usr/local/bin:/usr/bin:/bin:${NODEJS_HOME}/bin'

    // Credentials
//    GIT_CREDS = credentials('gitlab-user')
//    SIT_CREDS = credentials('sit-user')
//    UAT_CREDS = credentials('uat-user')

    // Variables
    NEXUS_URL = 'http://192.168.1.33:8085/nexus'
    PROJECT_NAME = 'econtract'
    SIT_ENV = '192.168.1.131'
    UAT_ENV = '192.168.1.131'

  }

  stages {

    ////////// Step 1 //////////
    stage('Git clone and Setup') {
      steps {
        script {
          sh(returnStdout: false, script: "git config --global credential.helper 'cache --timeout 86400'")
          sh(returnStdout: false, script: "git clean -f && git reset --hard origin/${BRANCH_NAME}")
          GIT_HEAD = sh(returnStdout: true, script: 'git rev-parse --short HEAD')

          echo "GIT_HEAD is ${GIT_HEAD}"
          branch = "${env.BRANCH_NAME}"
          releasedVersion = getReleaseVersion()
          echo "Building version ${env.BUILD_NUMBER} so released version is ${releasedVersion}"
          currentTime = sh(returnStdout: true, script: 'date +%Y-%m-%d').trim()
        }
      }
    }

    ////////// Step 2 //////////
    stage('Build and Unit tests') {
      when {
        allOf {
          expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
        }
      }
      steps {
        script {
          echo "Build package and Unit tests"
          nodejs('nodejs11.13') {
            sh "npm install --registry=${NEXUS_URL}/repository/npm-group/"
            sh "zip -r ${PROJECT_NAME}.zip app.js node_modules package-lock.json package.json views"
          }
        }
      }
    }

    ////////// Step 3 //////////
    stage('Upload Binary to Nexus') {
      when {
        allOf {
          expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
        }
      }
      steps {
        echo "Upload Binary to Nexus"
        sh "npm publish"
      }
    }

    ////////// Step 4 //////////
    stage('Deploy to SIT') {
      when {
        allOf {
          expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
        }
      }
      steps {
        script {
          echo "Deploy to SIT"
          sh "ssh ${SIT_CREDS_USR}@${SIT_ENV} cp -r /home/${SIT_CREDS_USR}/${PROJECT_NAME} /home/${SIT_CREDS_USR}/${PROJECT_NAME}-${currentTime}"
          sh "ssh ${SIT_CREDS_USR}@${SIT_ENV} rm -rf /home/${SIT_CREDS_USR}/${PROJECT_NAME}"
          sh "ssh ${SIT_CREDS_USR}@${SIT_ENV} mkdir -p /home/${SIT_CREDS_USR}/${PROJECT_NAME}"
          sh "scp -r ${PROJECT_NAME}.zip ${SIT_CREDS_USR}@${SIT_ENV}:/home/${SIT_CREDS_USR}/"
          sh "ssh ${SIT_CREDS_USR}@${SIT_ENV} unzip /home/${SIT_CREDS_USR}/${PROJECT_NAME}.zip -d /home/${SIT_CREDS_USR}/${PROJECT_NAME}"
          sh "ssh ${SIT_CREDS_USR}@${SIT_ENV} node /home/${SIT_CREDS_USR}/${PROJECT_NAME}/app.js &"
        }
      }
    }

    ////////// Step 5 //////////
    stage('Deploy to UAT') {
      when {
        allOf {
          expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
        }
      }
      steps {
        echo "Deploy to UAT"
        sh "ssh ${UAT_CREDS_USR}@${UAT_ENV} cp -r /home/${UAT_CREDS_USR}/${PROJECT_NAME} /home/${UAT_CREDS_USR}/${PROJECT_NAME}-${currentTime}"
        sh "ssh ${UAT_CREDS_USR}@${UAT_ENV} rm -rf /home/${UAT_CREDS_USR}/${PROJECT_NAME}"
        sh "ssh ${UAT_CREDS_USR}@${UAT_ENV} mkdir -p /home/${UAT_CREDS_USR}/${PROJECT_NAME}"
        sh "scp -r ${PROJECT_NAME}.zip ${UAT_CREDS_USR}@${UAT_ENV}:/home/${SIT_CREDS_USR}/"
        sh "ssh ${UAT_CREDS_USR}@${UAT_ENV} unzip /home/${UAT_CREDS_USR}/${PROJECT_NAME}.zip -d /home/${UAT_CREDS_USR}/${PROJECT_NAME}"
        sh "ssh ${UAT_CREDS_USR}@${UAT_ENV} node /home/${UAT_CREDS_USR}/${PROJECT_NAME}/app.js &"
      }
    }

  }

  post {
    always {
      deleteDir()
    }
    success {
      echo "success!!!"
//            emailext body: 'Success!!', recipientProviders: [developers()], subject: 'Success', to: 'admin@admin.com'
    }
    unstable {
      echo "unstable!!!"
//            emailext body: 'unstable!!', recipientProviders: [developers()], subject: 'unstable', to: 'admin@admin.com'
    }
    aborted {
      echo "aborted!!!"
//            emailext body: 'aborted!!', recipientProviders: [developers()], subject: 'aborted', to: 'admin@admin.com'
    }
    failure {
      echo "failure!!!"
//            emailext body: 'failure!!', recipientProviders: [developers()], subject: 'failure', to: 'admin@admin.com'
    }
  }
}

@NonCPS
def getReleaseVersion() {
  def packageJson = readJSON file: 'package.json'
  def versionNumber
  if (GIT_HEAD == null) {
    versionNumber = env.BUILD_NUMBER
  } else {
    versionNumber = GIT_HEAD.take(8).replace("\n", "")
  }

  return packageJson.version.concat("-${versionNumber}")
}
