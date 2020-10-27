properties([
  parameters ([
    string(name: 'DOCKER_REGISTRY_DOWNLOAD_URL',
           defaultValue: 'nexus-docker-private-group.ossim.io',
           description: 'Repository of docker images')
  ]),
  pipelineTriggers([
    [$class: "GitHubPushTrigger"]
  ]),
  [$class: 'GithubProjectProperty', displayName: '', projectUrlStr: 'https://github.com/Maxar-Corp/']
])

podTemplate(
  containers: [
    containerTemplate(
        name: 'docker',
        image: 'docker:19.03.11',
        ttyEnabled: true,
        command: 'cat',
        privileged: true
    ),
    containerTemplate(
        image: "${DOCKER_REGISTRY_DOWNLOAD_URL}/alpine/helm:3.2.3",
        name: 'helm',
        command: 'cat',
        ttyEnabled: true
    ),
    containerTemplate(
        name: 'git',
        image: 'alpine/git:latest',
        ttyEnabled: true,
        command: 'cat',
        envVars: [
            envVar(key: 'HOME', value: '/root')
        ]
    ),
    containerTemplate(
        image: "${DOCKER_REGISTRY_DOWNLOAD_URL}/docker-helper:1.0.0",
        name: 'docker-helper',
        command: 'cat',
        ttyEnabled: true
    )
  ],
  volumes: [
    hostPathVolume(
      hostPath: '/var/run/docker.sock',
      mountPath: '/var/run/docker.sock'
    ),
  ]
)
{
  node(POD_LABEL){
    stage("Checkout branch"){
        scmVars = checkout(scm)
        GIT_BRANCH_NAME = scmVars.GIT_BRANCH
        BRANCH_NAME = """${sh(returnStdout: true, script: "echo ${GIT_BRANCH_NAME} | awk -F'/' '{print \$2}'").trim()}"""
        VERSION = '1.0.1'
        ARTIFACT_NAME = 'unzip-and-ingest'
        GIT_TAG_NAME = ARTIFACT_NAME + "-" + VERSION
        script {
            if (BRANCH_NAME != 'master') {
                buildName "${VERSION}-SNAPSHOT - ${BRANCH_NAME}"
            } else {
                buildName "${VERSION} - ${BRANCH_NAME}"
            }
        }
    }

    stage("Load Variables"){
      step([$class     : "CopyArtifact",
            projectName: "gegd-dgcs-jenkins-artifacts",
            filter     : "common-variables.groovy",
            flatten    : true])
      load "common-variables.groovy"

      switch (BRANCH_NAME) {
        case "master":
          TAG_NAME = VERSION
          break
        case "dev":
          TAG_NAME = "latest"
          break
        default:
          TAG_NAME = BRANCH_NAME
          break
      }

      DOCKER_IMAGE_PATH = "${DOCKER_REGISTRY_PRIVATE_UPLOAD_URL}/"
    }

    stage("Build Docker Image") {
      container('docker-helper'){
        withDockerRegistry(credentialsId: 'dockerCredentials', url: "https://${DOCKER_REGISTRY_DOWNLOAD_URL}") {
          withGradle {
            script {
              sh './gradlew dockerPush'
            }
          }
        }
      }
    }

    stage('Tag Repo') {
      when (BRANCH_NAME == 'master') {
        container('git') {
          withCredentials([sshUserPrivateKey(
          credentialsId: env.GIT_SSH_CREDENTIALS_ID,
          keyFileVariable: 'SSH_KEY_FILE',
          passphraseVariable: '',
          usernameVariable: 'SSH_USERNAME')]) {
            script {
                sh """
                  mkdir ~/.ssh
                  echo -e "StrictHostKeyChecking=no\nIdentityFile ${SSH_KEY_FILE}" >> ~/.ssh/config
                  git config user.email "radiantcibot@gmail.com"
                  git config user.name "Jenkins"
                  git tag -a "${GIT_TAG_NAME}" \
                    -m "Generated by: ${env.JENKINS_URL}" \
                    -m "Job: ${env.JOB_NAME}" \
                    -m "Build: ${env.BUILD_NUMBER}"
                  git push -v origin "${GIT_TAG_NAME}"
                """
            }
          }
        }
      }
    }
  }
}
