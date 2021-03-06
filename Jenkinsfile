pipeline {
    agent {
      label "jenkins-gradle"
    }
    environment {
      ORG               = 'yahavi'
      APP_NAME          = 'spring-boot-http-gradle'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
      GRADLE_HOME       = "/opt/gradle/bin/gradle"
      JAVA_HOME         = "/usr/lib/jvm/java"
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          configArtifactory()
          container('gradle') {
            // TODO
            //sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
            runGradle()
            sh 'export VERSION=$PREVIEW_VERSION && skaffold run -f skaffold.yaml'
            sh "jx step validate --min-jx-version 1.2.36"
            sh "jx step post build --image \$JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:\$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }

          dir ('./charts/preview') {
           container('gradle') {
             sh "make preview"
             sh "jx preview --app $APP_NAME --dir ../.."
           }
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          configArtifactory()
          container('gradle') {
            // ensure we're not on a detached head
            sh "git checkout master"
            sh "git config --global credential.helper store"
            sh "jx step validate --min-jx-version 1.1.73"
            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            // TODO
            //sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
          }
          dir ('./charts/spring-boot-http-gradle') {
            container('gradle') {
              sh "make tag"
            }
          }
          container('gradle') {
            runGradle()

            sh 'export VERSION=`cat VERSION` && skaffold run -f skaffold.yaml'
            sh "jx step validate --min-jx-version 1.2.36"
            sh "jx step post build --image \$JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:\$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/spring-boot-http-gradle') {
            container('gradle') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'

              // release the helm chart
              sh 'make release'

              // promote through all 'Auto' promotion Environments
              sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
            }
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
    }
  }

def rtServer, buildInfo, rtGradle
void configArtifactory() {
    script {
      if (useArtifactory()) {
        rtServer = Artifactory.server "JX_ARTIFACTORY_SERVER"
        buildInfo = Artifactory.newBuildInfo()
        rtGradle = Artifactory.newGradleBuild()
        rtGradle.deployer repo: 'libs-release-local', server: rtServer
        rtGradle.resolver repo: 'libs-release', server: rtServer
        rtGradle.deployer.deployArtifacts = false
      }
    }
}

void runGradle() {
    script {
      if (useArtifactory()) {
        rtGradle.run rootDir: './', buildFile: 'build.gradle', tasks: 'clean build artifactoryPublish --no-daemon', buildInfo: buildInfo
        rtGradle.deployer.deployArtifacts buildInfo
        rtServer.publishBuildInfo buildInfo
      } else {
        sh "gradle clean build"
      }
    }
}

boolean useArtifactory() {
    script {
      try {
        return Artifactory != null && Artifactory.server("JX_ARTIFACTORY_SERVER") != null
      } catch (ignored) {
        return false
      }
    }
}
