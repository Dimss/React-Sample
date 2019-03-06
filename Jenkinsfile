import groovy.json.JsonOutput

def getJobName() {
    def jobNameList = env.JOB_NAME.split("/")
    if (jobNameList.size() > 0) {
        return jobNameList[jobNameList.size() - 1]
    } else {
        return jobName

    }
}

def getAppName() {
    if (env.gitlabActionType == "TAG_PUSH") {
        return "${getJobName()}-${getGitTag()}".replaceAll("\\.","-")
    } else {
        return "${getJobName()}-${getGitCommitShortHash()}".replaceAll("\\.","-")
    }
}

def getGitCommitShortHash() {
    return checkout(scm).GIT_COMMIT.substring(0, 7)
}

def getGitTag() {
    if (env.gitlabActionType == "TAG_PUSH") {
        def tagPathList = env.gitlabSourceBranch.split("/")
        return tagPathList[tagPathList.size() - 1]
    } else {
        return ""
    }
}

def getApiUrl() {
    def profile = getProfile()
    if(profile == "dev")
      return "http://${getRoutePrefix()}.router.default.svc.cluster.local/api/Person"
    if(profile == "lab")
      return "http://${getRoutePrefix()}.router.default.svc.cluster.local/api/Person"
}

def getRoutePrefix(){
  if(profile == "dev")
    return "coreapitest-dev-latest"
  if(profile == "lab")
    return "coreapitest-lab-latest"
}

def getDockerImageTag() {
    if (env.gitlabActionType == "TAG_PUSH") {
        return getGitTag()
    } else {
        return "${getGitCommitShortHash()}-${currentBuild.number}"
    }
}

def getProfile(){
    if (env.gitlabActionType == "TAG_PUSH") {
        return "lab"
    } else {
        return "dev"
    }
}

pipeline {
    agent any
    stages {
        stage('Checkout GIT Tag (in case it was pushed) ') {

            steps {
                script {
                    if (env.gitlabActionType == "TAG_PUSH") {
                        checkout poll: false, scm: [
                                $class                           : 'GitSCM',
                                branches                         : [[name: "${env.gitlabSourceBranch}"]],
                                doGenerateSubmoduleConfigurations: false,
                                submoduleCfg                     : [],
                        ]
                    }
                }
            }
        }


        stage("Create S2I image stream and build configs") {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def icBcTemplate = readFile('ocp/ci/app-is-bc.yaml')
                            def models = openshift.process(icBcTemplate,
                                    "-p=BC_IS_NAME=${getAppName()}",
                                    "-p=DOCKER_REGISTRY=${env.DOCKER_REGISTRY}",
                                    "-p=DOCKER_IMAGE_NAME=/${env.DOCKER_IMAGE_PREFIX}/${GOVIL_APP_NAME}",
                                    "-p=DOCKER_IMAGE_TAG=${getDockerImageTag()}",
                                    "-p=GIT_REPO=${scm.getUserRemoteConfigs()[0].getUrl()}",
                                    "-p=GIT_REF=${env.gitlabSourceBranch}",
                                    "-p=S2I_BUILDER_ISTAG=${env.S2I_BUILD_IMAGE}",
                                    "-p=API_URL=${getApiUrl()}"
                            )
                            echo "${JsonOutput.prettyPrint(JsonOutput.toJson(models))}"
                            openshift.create(models)
                            def bc = openshift.selector("buildconfig/${getAppName()}")
                            def build = bc.startBuild()
                            build.logs("-f")
                            openshift.delete(models)
                        }
                    }
                }
            }
        }

        stage("Deploy to OpenShift") {
          steps{
            script{
              openshift.withCluster() {
                openshift.withProject() {
                  def size = 1
                  def appName = getAppName()
                  def namespace = openshift.project()
                  def port = 8080
                  def image = "${env.DOCKER_REGISTRY}/${env.DOCKER_IMAGE_PREFIX}/${GOVIL_APP_NAME}:${getDockerImageTag()}"
                  def profile = getProfile()
                  def routeName = "reactsample-dev"
                  def crTemplate = readFile('ocp/cd/cr-template.yaml')
                  def models = openshift.process(crTemplate,
                        "-p=SIZE=${size}",
                        "-p=APP_NAME=${appName}",
                        "-p=NAMESPACE=${namespace}",
                        "-p=IMAGE=${image}",
                        "-p=PORT=${port}",
                        "-p=ROUTE_PREFIX=${getRoutePrefix()}",
                        "-p=PROFILE=${profile}")
                  echo "${JsonOutput.prettyPrint(JsonOutput.toJson(models))}"
                  openshift.create(models)
                }
              }
            }
          }
        }
    }
}
