import groovy.json.JsonOutput

/** Tox environment */
def environment = 'tests'

/** Run Tox
 *
 * @param environment test environment to run
*/
def runTox(environment) {
  def processes = env.PYTEST_PROCESSES ?: 'auto'
  try {
    wrap([$class: 'AnsiColorBuildWrapper']) {
      withEnv(["PYTEST_ADDOPTS=${PYTEST_ADDOPTS} " +
        "-n=${processes} " +
        "--color=yes"]) {
        sh "tox -e ${environment}"
      }
    }
} catch(err) {
    currentBuild.result = 'FAILURE'
    throw err
  } finally {
    dir('results') {
      stash environment
    }
  }
}

/** Send a notice to #fxtest-alerts on irc.mozilla.org with the build result
 *
 * @param result outcome of build
*/
def ircNotification(result) {
  def nick = "fxtest${BUILD_NUMBER}"
  def channel = '#fx-test-alerts'
  result = result.toUpperCase()
  def message = "Project ${JOB_NAME} build #${BUILD_NUMBER}: ${result}: ${BUILD_URL}"
  node {
    sh """
        (
        echo NICK ${nick}
        echo USER ${nick} 8 * : ${nick}
        sleep 5
        echo "JOIN ${channel}"
        echo "NOTICE ${channel} :${message}"
        echo QUIT
        ) | openssl s_client -connect irc.mozilla.org:6697
    """
  }
}

stage('Checkout') {
  node {
    timestamps {
      deleteDir()
      checkout scm
      stash name: 'workspace'
    }
  }
}

stage('Lint') {
  node {
    timestamps {
      deleteDir()
      unstash 'workspace'
      sh 'tox -e flake8'
    }
  }
}

try {
  stage('Test') {
    node {
      timeout(time: 5, unit: 'MINUTES') {
        timestamps {
          deleteDir()
          unstash 'workspace'
          try {
            runTox(environment)
          } catch(err) {
            currentBuild.result = 'FAILURE'
            throw err
          } finally {
            dir('results') {
              stash environment
            }
          }
        }
      }
    }
  }    
} catch(err) {
  currentBuild.result = 'FAILURE'
  ircNotification(currentBuild.result)
  mail(
    body: "${BUILD_URL}",
    from: "firefox-test-engineering@mozilla.com",
    replyTo: "firefox-test-engineering@mozilla.com",
    subject: "Build failed in Jenkins: ${JOB_NAME} #${BUILD_NUMBER}",
    to: "fte-ci@mozilla.com")
  throw err
} finally {
  stage('Results') {
    node {
      deleteDir()
      sh 'mkdir results'
      dir('results') {
        unstash environment
      }
      publishHTML(target: [
        allowMissing: false,
        alwaysLinkToLastBuild: true,
        keepAll: true,
        reportDir: 'results',
        reportFiles: "${environment}.html",
        reportName: 'HTML Report'])
      junit 'results/*.xml'
      archiveArtifacts 'results/*'
    }
  }
}
