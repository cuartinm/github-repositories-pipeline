#!/user/bin/env groovy

withCredentials([string(credentialsId: 'GH_GENERIC_WEBHOOK_TOKEN', variable: 'GH_GENERIC_WEBHOOK_TOKEN')]) {       
  properties([
    pipelineTriggers([
      [$class: 'GenericTrigger',
        genericVariables: [
          [key: 'clone_url', value: '$.repository.clone_url'],
          [key: 'action', value: '$.action'],
          [key: 'head_branch', value: '$.pull_request.head.ref'],
          [key: 'statuses_url', value: '$.pull_request.statuses_url'],
          [key: 'base_branch', value: '$.pull_request.base.ref'],
          [key: 'merged', value: '$.pull_request.merged']
        ],
        token: "$GH_GENERIC_WEBHOOK_TOKEN",
        causeString: "Triggered on $action pull request",
        printPostContent: false,
        printContributedVariables: false,
      ]
    ])
  ])
}

node {
  def error = null
  def target_branch = ""
  def pull_request = true

  switch("$action") {
    case "opened":
      target_branch = "$head_branch"
      break
    case "closed":
      target_branch = "$base_branch"
      break
    default:
      pull_request = false
      break
  }
  
  try {
    if (pull_request) {
      checkout("$clone_url", target_branch)
      init()
      validate()
      plan()
      if("$merged".toBoolean()) {
        apply()
      }
      setGitHubStatus("continuous-integration", "success", "your Job was successful. You can check your logs in the following link ->", "${env.RUN_DISPLAY_URL}")
    }
  } catch(Exception e) {
    currentBuild.result = 'FAILURE'
    echo "Exception: ${e}"
    setGitHubStatus("continuous-integration", "failure", "Your job failed. Please check your logs in the following link ->", "${env.RUN_DISPLAY_URL}")
  } finally {
    notifyBuild(currentBuild.result)
    cleanWs()
  }
}


def setGitHubStatus(context, state, description, target_url){
  withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {

    def builder = new groovy.json.JsonBuilder()
    builder context: "$context", state: "$state", description: "$description", target_url: "$target_url"
    try {
      def httpConn = new URL("$statuses_url").openConnection();
      httpConn.setRequestMethod("POST");
      httpConn.setRequestProperty("Authorization", "token $GITHUB_ACCESS_TOKEN")
      httpConn.setRequestProperty("Accept", "application/vnd.github.v3+json")
      httpConn.setRequestProperty("Accept", "application/json");
      httpConn.setDoOutput(true);
      httpConn.getOutputStream().write(builder.toString().getBytes("UTF-8"));
      return httpConn.getResponseCode();
    } catch(Exception e){
      echo "Exception: ${e}"
    }           
  }
}

def checkout(repo, branch) {
  stage('Checkout') {
    checkout([
      $class: 'GitSCM',
      branches: [[name: branch]],
      doGenerateSubmoduleConfigurations: false,
      extensions: [[
        $class: 'CloneOption',
        noTags: false,
        reference: '',
        shallow: false
      ]],
      submoduleCfg: [],
      userRemoteConfigs: [[
        url: repo
      ]]
    ])
  }
}

def init() {
  stage('Init') {
    try {
      def plan_command = sh(script: "terraform init", returnStatus: true)
    } catch(Exception e) {
      currentBuild.result = 'FAILURE'
      echo "Exception: ${e}"
    }
  }
}

def validate() {
  stage('Validate') {
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {
      try {
        def plan_command = sh(script: "terraform validate", returnStatus: true)
      } catch(Exception e) {
        currentBuild.result = 'FAILURE'
        echo "Exception: ${e}"
      }
    }
  }
}

def plan() {
  stage('Plan') {
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {
      try {
        def plan_command = sh(script: "terraform plan -var='github_token=$GITHUB_ACCESS_TOKEN'", returnStatus: true)
      } catch(Exception e) {
        currentBuild.result = 'FAILURE'
        echo "Exception: ${e}"
      }
    }
  }
}

def apply() {
  stage('Apply') {
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {
      try {
        def plan_command = sh(script: "terraform apply -var='github_token=$GITHUB_ACCESS_TOKEN' -auto-approve", returnStatus: true)
      } catch(Exception e) {
        currentBuild.result = 'FAILURE'
        echo "Exception: ${e}"
      }
    }
  }
}

def notifyBuild(currentBuild = 'SUCCESS') {

}