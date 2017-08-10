#!groovy
def mvnDefaultOptions = '--batch-mode -V -U -e'
def pom

def userInput
    timeout(time: 10, unit: 'MINUTES') {
      userInput = input(message: 'Proceed with deployment?', parameters: [
              [$class: 'TextParameterDefinition', name: 'projectId', defaultValue: 'iva-poc-dev', description: 'The GCP project id environment you want to deploy to (e.g. iva-poc-dev, iva-poc-stage, iva-prod...)'],
              [$class: 'TextParameterDefinition', name: 'namespace', defaultValue: 'dev', description: 'The namespace to be used by this version (e.g. dev, stage, prod...)'],
              [$class: 'TextParameterDefinition', name: 'node', defaultValue: 'jenkins-lnx-slave3', description: 'If the deployment should be promoted and live (true) or not (false)']
      ])
      nodeid='${userInput.node}'     
      echo ("projectId: "+userInput['projectId'])  
      echo ("namespace: "+userInput['namespace']) 
      echo ('${userInput.node}')
      
    }

node ('${userInput.node}') {
  stage('Ready to deploy?') {
    
      echo ("projectId: "+userInput['projectId'])  
      echo ("namespace: "+userInput['namespace']) 
      echo ("node: "+userInput['node']) 
    }
 }
