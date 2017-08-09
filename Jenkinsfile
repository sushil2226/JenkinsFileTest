#!groovy
def mvnDefaultOptions = '--batch-mode -V -U -e'
def pom

node ('jenkins-lnx-slave3'){
  try {
    withEnv(["JAVA_HOME=${tool 'JDK-1.7-latest (1.7.0_80-b15)'}", "PATH+MAVEN=${tool 'M3'}/bin:${env.JAVA_HOME}/bin"]) {

      step([$class: 'WsCleanup'])

      stage('Checkout') {
        checkout scm
        pom = readMavenPom()
        echo "Building ${pom.artifactId} version ${pom.version} from branch ${env.BRANCH_NAME}..."
      }

      stage('Compile') {
        milestone()
        sh "mvn ${mvnDefaultOptions} clean compile -DskipTests"
      }

      stage('Unit Tests') {
        milestone()
        def ignoreTestFailure = defineCriticalBranch(env.BRANCH_NAME) != null ? false : true;
        sh "mvn ${mvnDefaultOptions} test verify -P all-tests -Dmaven.test.failure.ignore=${ignoreTestFailure}"
        junit 'target/surefire-reports/*.xml'
      }

      stage('Quality Analysis') {
        milestone()
        def sonarBranch = defineSonarBranch(env.BRANCH_NAME);

        if (sonarBranch != null) {
          // Only run analisys on critical branches
          withEnv(["JAVA_HOME=${tool 'JDK-1.8-latest (1.8.0_66-b17)'}", "PATH+MAVEN=${tool 'M3'}/bin:${env.JAVA_HOME}/bin"]) {
            withSonarQubeEnv {
              try {
                sh """mvn ${mvnDefaultOptions} sonar:sonar \
                                    -Dsonar.branch=${sonarBranch} \
                                    -Dsonar.sources=src/main/java \
                                    -Dsonar.tests=src/test/java \
                                    -Dsonar.java.binaries=target/${pom.artifactId}-${pom.version}/WEB-INF/classes \
                                    -Dsonar.coverage.exclusions=**/endpoint/**/*,**/entity/**/*,**/to/**/*,**/filter/**/*,**/taskhandler/**/*,**/StorageService*,**/servlet/** \
                                    -Dsonar.junit.reportsPath=target/surefire-reports"""
              } catch (err) {
                echo "Sonar analisys failed, but the build should continue. Error: ${err}"
              }
            }
          }
        }

        stash includes: '**', name: 'app'
      }
    }

    currentBuild.result = 'SUCCESS'
  } catch (e) {
    currentBuild.result = 'FAILURE'
    notifyBuildStatus()
    throw e
  }
}

node ('jenkins-lnx-slave3'){
  stage('Ready to deploy?') {

    def userInput
    def shouldNotifyWaitingApproval = defineCriticalBranch(env.BRANCH_NAME) != null ? false : true;

    if (shouldNotifyWaitingApproval) {
      emailext(
        subject: "Build ${pom.artifactId} version ${pom.version} on branch ${env.BRANCH_NAME} waiting deployment approval",
        body: """
        Build ${pom.artifactId} version ${pom.version} on branch ${env.BRANCH_NAME} waiting deployment approval.
        Job: ${env.JOB_NAME} [${env.BUILD_NUMBER}]
        Click here to approve/cancel deployment: ${env.BUILD_URL}input
        """,
        to: getEmailRecipients()
      )
    }

    timeout(time: 10, unit: 'MINUTES') {
      userInput = input(message: 'Proceed with deployment?', parameters: [
              [$class: 'TextParameterDefinition', name: 'projectId', defaultValue: 'iva-poc-dev', description: 'The GCP project id environment you want to deploy to (e.g. iva-poc-dev, iva-poc-stage, iva-prod...)'],
              [$class: 'TextParameterDefinition', name: 'namespace', defaultValue: 'dev', description: 'The namespace to be used by this version (e.g. dev, stage, prod...)'],
              [$class: 'TextParameterDefinition', name: 'promoteDeploy', defaultValue: 'true', description: 'If the deployment should be promoted and live (true) or not (false)']
      ])
    }

    try {
      node ('jenkins-lnx-slave3'){
        unstash 'app'
        pom = readMavenPom()
        applicationVersion = defineApplicationVersion(userInput.namespace, pom)
        echo "Deploying using this parameters: userInput=${userInput}, applicationVersion=${applicationVersion}"

        withEnv(["JAVA_HOME=${tool 'JDK-1.7-latest (1.7.0_80-b15)'}", "PATH+MAVEN=${tool 'M3'}/bin:${env.JAVA_HOME}/bin"]) {
          stage('Deploy') {
            // copy the corresponding environment service account credential file
            def serviceAccountCredentialPath = './src/main/webapp/WEB-INF'

            def command = defineCommanServiceAccount(env.serviceaccountpath);

            sh """ ${command} ${env.serviceaccountpath} \
                  ${serviceAccountCredentialPath}/iva-api-service-account.json"""

            // mvn clean package gcloud:deploy -Dpromote-deploy=true -Dgcp-project.id='iva-poc-dev' -Dapplication.version='dev'
            sh """mvn ${mvnDefaultOptions} package gcloud:deploy -DskipTests \
                  -Dpromote-deploy='${userInput.promoteDeploy}' \
                  -Dgcp-project.id='${userInput.projectId}' \
                  -Dapplication.version='${applicationVersion}'"""
            archiveArtifacts artifacts: 'target/*.war', fingerprint: true, onlyIfSuccessful: true
          }
        }

        currentBuild.result = 'SUCCESS'
      }
    } catch (e) {
      currentBuild.result = 'FAILURE'
      throw e
    } finally {
      notifyBuildStatus()
    }
  }
}

def notifyBuildStatus() {
  try {
    if (currentBuild.result == 'SUCCESS') {
      notifySuccessful()
    } else {
      notifyFailed()
    }
  } catch (e) {
    echo "Failed notifying about the build status."
  }
}

def notifySuccessful() {
  if (defineCriticalBranch(env.BRANCH_NAME) != null) {
    // only notifies on critical branches!
    step([$class                  : 'Mailer',
          notifyEveryUnstableBuild: true,
          recipients              : getEmailRecipients(),
          sendToIndividuals       : true])
  }
}

def notifyFailed() {
  step([$class                  : 'Mailer',
        notifyEveryUnstableBuild: true,
        recipients: getEmailRecipients(),
        sendToIndividuals       : true])
}

def getEmailRecipients() {
  return ''
}

def defineCriticalBranch(String branchName) {
  def branchParam = null;

  if (branchName == "master"
          || branchName == "develop"
          || branchName.startsWith("hotfix")
          || branchName.startsWith("bugfix")
          || branchName.startsWith("release")) {
    branchParam = branchName
  }

  return branchParam
}

def defineSonarBranch(String branchName) {
  def branchParam = null;

  if (branchName == "master"
          || branchName == "develop"
          || branchName.startsWith("release")) {
    branchParam = branchName
  }

  return branchParam
}

def defineApplicationVersion(namespace, pom) {
  return namespace + "-iva-api-" + pom.version.replaceAll("\\.", "-").toLowerCase()
}

def defineCommanServiceAccount(String serviceaccountpath) {
  def command =  null;

  if(serviceaccountpath.startsWith("gs:") ) {
    command = "gsutil cp"
  } else {
    command = "cp"
  }

  return command
}
