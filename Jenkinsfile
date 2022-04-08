pipeline {
  agent {
    kubernetes {
        inheritFrom 'centos-7'
    }
  }

  triggers {
    // only on master branch / 30 minutes before nightly sign-and-deploy
    cron(env.BRANCH_NAME == 'master' ? 'H 15 * * *' : '')
    githubPush()
  }

  environment {
    GRADLE_USER_HOME = "$WORKSPACE/.gradle" // workaround for https://bugs.eclipse.org/bugs/show_bug.cgi?id=564559
  }

  options {
    buildDiscarder(logRotator(numToKeepStr:'15'))
    disableConcurrentBuilds()
    timeout(time: 90, unit: 'MINUTES')
  }

  // Build stages
  stages {
    stage('Build') {
      tools {
        maven "apache-maven-3.8.5"
        jdk "temurin-jdk11-latest"
      }
      environment {
        EXTRA_ARGS = "-Dmaven.repo.local=.m2 -Dtycho.localArtifacts=ignore"
        MAVEN_OPTS="-Xmx512m"
      }
      steps {
        checkout scm
        wrap([$class: 'Xvnc', takeScreenshot: false, useXauthority: true]) {
          sh 'mvn clean install'
        }
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'org.eclipse.xtext.orbitreplacement.repository/target/repository/**'
    }
    cleanup {
      script {
        def curResult = currentBuild.currentResult
        def lastResult = 'NEW'
        if (currentBuild.previousBuild != null) {
          lastResult = currentBuild.previousBuild.result
        }

        if (curResult != 'SUCCESS' || lastResult != 'SUCCESS') {
          def color = ''
          switch (curResult) {
            case 'SUCCESS':
              color = '#00FF00'
              break
            case 'UNSTABLE':
              color = '#FFFF00'
              break
            case 'FAILURE':
              color = '#FF0000'
              break
            default: // e.g. ABORTED
              color = '#666666'
          }

          slackSend (
            message: "${lastResult} => ${curResult}: <${env.BUILD_URL}|${env.JOB_NAME}#${env.BUILD_NUMBER}>",
            botUser: true,
            channel: 'xtext-builds',
            color: "${color}"
          )
        }
      }
    }
  }
}