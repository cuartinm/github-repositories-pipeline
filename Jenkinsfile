#!/user/bin/env groovy

library 'jenkins-library@master'

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
        regexpFilterExpression: '^((false (opened|reopened|synchronize))|(true (closed)))? (develop|master)?$',
        regexpFilterText: '$merged $action $base_branch'
      ]
    ])
  ])
}

node {

  def target_branch = "$head_branch"

  if ("$action" == "closed") {
    target_branch = "$base_branch"
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
      withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {       
        def code = gitHubStatus(
                    context: "continuous-integration",
                    state: "success",
                    description: "your Job was successful. You can check your logs in the following link ->",
                    target_url: "${env.RUN_DISPLAY_URL}",
                    github_access_token: "$GITHUB_ACCESS_TOKEN",
                    statuses_url: "$statuses_url"
                    )
      }    
    }
  } catch(Exception e) {
    currentBuild.result = 'FAILURE'
    echo "Exception: ${e}"
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {       
      def code = gitHubStatus(
                  context: "continuous-integration",
                  state: "failure",
                  description: "Your job failed. Please check your logs in the following link ->",
                  target_url: "${env.RUN_DISPLAY_URL}",
                  github_access_token: "$GITHUB_ACCESS_TOKEN",
                  statuses_url: "$statuses_url"
                  )
    }
  } finally {
    notifyBuild(currentBuild.result)
    cleanWs()
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
    def plan_command = sh(script: "terraform init", returnStatus: true)
  }
}

def validate() {
  stage('Validate') {
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {
      def plan_command = sh(script: "terraform validate", returnStatus: true)
    }
  }
}

def plan() {
  stage('Plan') {
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {
      def plan_command = sh(script: "terraform plan -var='github_token=$GITHUB_ACCESS_TOKEN'", returnStatus: true)
    }
  }
}

def apply() {
  stage('Apply') {
    withCredentials([string(credentialsId: 'GITHUB_ACCESS_TOKEN', variable: 'GITHUB_ACCESS_TOKEN')]) {
      def plan_command = sh(script: "terraform apply -var='github_token=$GITHUB_ACCESS_TOKEN' -auto-approve", returnStatus: true)
    }
  }
}