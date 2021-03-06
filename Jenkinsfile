import groovy.json.JsonSlurperClassic

String GIT_LOG;
String GIT_LOCAL_BRANCH;

def caughtError;

node(label: 'Mobile Builder 2') {
  currentBuild.result = "SUCCESS"

  ansiColor('xterm') {
    try {

      stage('Checkout') {
         checkout scm
      }

      String PATH = "PATH=$PATH:/usr/local/bin"; // this is to find the npm command

      String LOCK_SCOPE = env.NODE_NAME.replace(/ /, '-')
      lock(resource: "mobile-web-performance-lock-$LOCK_SCOPE") {
        stage('foo') {
          COMMIT_HASH = sh (
            script: "git ls-remote --heads https://github.com/Workable/workable.git | grep \$BRANCH\$ | awk '{print \$1}'",
            returnStdout: true).trim()
          INDICATIVE_RESULTS = "139882596 3124 2921"
          echo "$BRANCH:$COMMIT_HASH"
          currentBuild.description = COMMIT_HASH
          echo "currentBuild.description: ${currentBuild.description}"
          echo "previousBuild.description: ${currentBuild.previousBuild.description}"
        }
      }

    } catch (err) {
      caughtError = err;
      currentBuild.result = "FAILURE"
    } finally {
      switch(currentBuild.result) {
        case "FAILURE":
          slackit([
            channel: YODA_SLACK_CHANNEL,
            color: "danger",
            message: "${JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)\n```${BRANCH}```${isAbortException ? '' : '\n```' + caughtError + '```'}"
          ])
          if (caughtError) {
           throw caughtError; // rethrow so that the build fails
          }
          break;
        default:
          resultsMap = INDICATIVE_RESULTS.tokenize(' ')
          sheetId = resultsMap[0]
          basisBranchTime = resultsMap[1]
          branchTime = resultsMap[2]

          result = (basisBranchTime > branchTime) ? "faster" : "slower"
          diff = percentage(result, basisBranchTime.toInteger(), branchTime.toInteger())

          Integer[] testArray = [BRANCH.length(), BASIS_BRANCH.length()]
          paddingLength = testArray.max() + 5

          slackit([
            channel: YODA_SLACK_CHANNEL,
            color: (result == "faster") ? "good" : "warning",
            message: """
                    |${JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)\n
                    |```
                    |${BASIS_BRANCH} - ${BRANCH} (${COMMIT_HASH})
                    |
                    |== DASHBOARD ==
                    |${(BASIS_BRANCH+":").padRight(paddingLength)} ${basisBranchTime}ms
                    |${(BRANCH+":").padRight(paddingLength)} ${branchTime}ms - ${diff}x ${result}
                    |```
                    |\n
                    |<https://docs.google.com/spreadsheets/d/${YODA_SHEET_ID}#gid=${sheetId}|View Results>
                    """.stripMargin()
          ])
      }
    }
  }
}

def percentage(String result, int previous, int after) {
  if (result == "faster") {
    diff = 1 - (after / previous)
  } else {
    diff = 1 - (previous / after)
  }
  return "${diff.toString().substring(0, 4)}"
}

def slackit(params) {
  if (!env.SKIP_SLACK) {
    slackSend(params)
  }
}