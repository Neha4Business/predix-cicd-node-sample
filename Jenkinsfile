#!groovy

//This example is a single branch pipeline for use in the Predix CI/CD Platform

def cf1_api = 'https://api.system.aws-usw02-pr.ice.predix.io'
def cf3_api = 'https://api.system.aws-usw02-dev.ice.predix.io'

try {
    node('predixci-node6.9') {
        def server = Artifactory.server('R2-artifactory')

        def branchName = env.BRANCH_NAME
        def shortCommit

        stage('Checkout') {
            echo "Checking out ${branchName}"
            checkout scm
        }

        stage('Build') {
            // Most typical, if you're not cloning into a sub directory
            gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
            // short SHA, possibly better for chat notifications, etc.
            shortCommit = gitCommit.take(6)
            echo shortCommit

            echo 'Installing dependencies'
            sh "npm config set registry ${server.getUrl()}/api/npm/npm-virtual" //Use Caching repo for faster build
            sh 'npm install'

            //Our project only requires npm install. Do additional steps here for your own project like grunt dist to packge your app

            echo 'Packaging artifact'
            sh "zip predix-cicd-node-sample-${env.BUILD_NUMBER}.zip *"

            stash includes: '*.zip', name: 'artifact'
            stash includes: 'manifest.yml', name: 'manifest'

            if ("master".equals(branchName)) { //Upload master artifact for release
            echo 'Deploying artifact to Artifactory'
            def uploadSpec = """{
                "files": [
                    {
                        "pattern": "*.zip",
                        "target": "libs-release-local/predixcicd-node-sample-${env.BUILD_NUMBER}/"
                    }
                ]
            }"""

            def buildInfo = server.upload(uploadSpec)
            server.publishBuildInfo(buildInfo)
          }
        }


            stage('Test') {
                sh 'npm test'
            }
    }

    node ("predixci-jdk-1.8") {
      stage("Deploy To Dev Using CF CLI") {
        unstash 'artifact'

        sh "unzip predix-cicd-node-sample-${env.BUILD_NUMBER}.zip"

        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'cf3_login',
                usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                cf_deploy(cf3_api, 'predix-devops', 'demo', USERNAME, PASSWORD, predix-cicd-node-sample-cf)
        }
      }
    }

    node ("predixci-pcd") {
        stage("Deploy To Dev Using PCD") {
            pcdOutput = sh(returnStatus: true, script: 'pcd')
            if (pcdOutput == 0)
            {
              echo "PCD tool is available"
              env = "predix-cf3" //Or predix-cf1
              org ="predix-devops"
              space = "demo"

              unstash 'artifact'
              unstash 'manifest'
              artifact_url = "predix-cicd-node-sample-${env.BUILD_NUMBER}.tar"
              manifest_url ="manifest.yml"
              version = "${env.BUILD_NUMBER}"
              app_name = "predix-cicd-node-sample"

              withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'cf3_token',
                      usernameVariable: 'USERNAME', passwordVariable: 'token']]) {
                      pcd_deploy(env, org, space, USERNAME, token, "${artifact_url}","${manifest_url}","${version}","${app_name}");
              }
            }
            else {
                echo "PCD tool not found"
            }
        }
    }

    if ("master".equals(branchName)) {
        waitForApprovalStaging();

        node ("predixci-jdk-1.8") {
          stage("Deploy To Stage Using CF CLI") {
            unstash 'artifact'

            sh "unzip predix-cicd-node-sample-${env.BUILD_NUMBER}.zip"

            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'cf1_login',
                    usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                    cf_deploy(cf1_api, 'predix-devops-runtime', 'stage', USERNAME, PASSWORD, 'predix-cicd-node-sample-cf')
            }
          }
        }
        node ("predixci-pcd"){
            stage("Deploy to Stage Using PCD") {
                pcdOutput = sh(returnStatus: true, script: 'pcd')
                if(pcdOutput == 0)
                {
                  echo "PCD tool is available"
                  env = "predix-cf1" //Or predix-cf1
                  org ="predix-devops-runtime"
                  space = "stage"

                  unstash 'artifact'
                  unstash 'manifest'
                  artifact_url = "predix-cicd-node-sample-${env.BUILD_NUMBER}.tar"
                  manifest_url ="manifest.yml"
                  version = "${env.BUILD_NUMBER}"
                  app_name = "predix-cicd-node-sample"

                  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'cf3_token',
                          usernameVariable: 'USERNAME', passwordVariable: 'token']]) {
                          pcd_deploy(env, org, space, USERNAME, token, "${artifact_url}","${manifest_url}","${version}","${app_name}");
                  }
                }
                else{
                    echo "PCD tool not found"
                }
            }
        }

        node ("predixci-jdk-1.8"){
            stage("Integration test") {
                echo "integration test"
                sleep 10
                echo "integration test"
            }
            stage("Compliance test") {
                echo "Compliance test"
                sleep 10
                echo "Compliance test"
            }
        }
        waitForApprovalProduction();
        node ("predixci-jdk-1.8") {
          stage("Deploy To Prod Using CF CLI") {
            unstash 'artifact'

            sh "unzip predix-cicd-node-sample-${env.BUILD_NUMBER}.zip"

            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'cf1_login',
                    usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                    cf_deploy(cf1_api, 'predix-devops-runtime', 'prod', USERNAME, PASSWORD, 'predix-cicd-node-sample-cf')
            }
          }
        }
        node ("predixci-pcd"){
            stage("Deploy to Prod Using PCD") {
                pcdOutput = sh(returnStatus: true, script: 'pcd')
                if(pcdOutput == 0)
                {
                  env = "<Point of Presence Goes here (predix-cf1 or predix-cf3)>"
                  org ="<org_goes_here>"
                  space = "<space_goes_here>"
                  user_name = "<User_id_goes_here>"
                  token_id = "<token_goes_here>"

                  unstash 'artifact'
                  unstash 'manifest'
                  artifact_url = "<artifact_url_goes_here>"
                  manifest_url ="<manifest_url_goes_here>"
                  version = "<Version goes here>"
                  app_name = "<insert_app_name_here>-${env.BUILD_ID}"
                  pcd_deploy("${env}","${org}","${space}","${user_name}","${token_id}","${artifact_url}","${manifest_url}","${version}","${app_name}");
                }
                else{
                    echo "PCD tool not found"
                }
            }
        }
    }
} catch (exc) {
    echo "Caught: ${exc}"
}

def waitForApprovalStaging(){
    stage("Ready to go staging?") {
        timeout(time:1, unit:'DAYS') {
            input message:'Approve deployment to staging?', submitter: 'it-ops' //The submitter is the user that has the administrative power to approve deployment
        }
    }
}

def waitForApprovalProduction(){
    stage("Ready to go production?") {
        timeout(time:5, unit:'DAYS') {
            input message:'Approve deployment to production?', submitter: 'it-ops' //The submitter is the user that has the administrative power to approve deployment
        }
    }
}

//Deploy function that uses the CF CLI (CF push)
def cf_deploy(String api, String, org, String space, String user_name, String password, String app_name){
  sh "cf login -a '${api}' --skip-ssl-validation -u '${user_name}' -p '${password}' -o '${org}' -s '${space}'"

  sh "cf push '${app_name}'"
}

//Deploy function that uses the PCD CLI (Deployment Service)
def pcd_deploy(String env, String org, String space, String user_name, String token_id, String artifact_url, String manifest_url,String version, String app_name){
  sh 'pcd deploy auth -o ${org} -s${space} -u ${user_name} -tid ${token_id} -e ${env}'
  sh 'pcd deploy -a ${artifact_url} -m ${manifest_url} -v ${version} -n ${app_name} -hn ${app_name}' //-hn flag is for blue green deployment
}
