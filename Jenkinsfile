library identifier: "pipeline-library@master",
retriever: modernSCM(
  [
    $class: "GitSCMSource",
    remote: "https://github.com/redhat-cop/pipeline-library.git"
  ]
)

openshift.withCluster() {
  env.NAMESPACE = openshift.project()
  env.POM_FILE = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"
  env.APP_NAME = "${JOB_NAME}".replaceAll(/-build.*/, '')
  echo "Starting Pipeline for ${APP_NAME}..."
  env.BUILD = "${env.NAMESPACE}"
  env.DEV = "${APP_NAME}-dev"
  env.STAGE = "${APP_NAME}-stage"
  env.PROD = "${APP_NAME}-prod"
}

pipeline {
  // Use Jenkins Maven slave
  // Jenkins will dynamically provision this as OpenShift Pod
  // All the stages and steps of this Pipeline will be executed on this Pod
  // After Pipeline completes the Pod is killed so every run will have clean
  // workspace
  agent {
    label 'maven'
  }

  // Pipeline Stages start here
  // Requeres at least one stage
  stages {

    // Checkout source code
    // This is required as Pipeline code is originally checkedout to
    // Jenkins Master but this will also pull this same code to this slave
    stage('Git Checkout') {
      steps {
        // Turn off Git's SSL cert check, uncomment if needed
        // sh 'git config --global http.sslVerify false'
        //git url: "${APPLICATION_SOURCE_REPO}", branch: "${APPLICATION_SOURCE_REF}"
        git url: "https://github.com/jarekpc/simple-spring.git", branch: "master"
      }
    }

    // Run Maven build, skipping tests
    stage('Build'){
      steps {
        sh "mvn -B clean install -DskipTests=true -f ${POM_FILE}"
      }
    }

    // Run Maven unit tests
    stage('Unit Test'){
      steps {
        sh "mvn -B test -f ${POM_FILE}"
      }
    }

    // Build Container Image using the artifacts produced in previous stages
    stage('Build Container Image'){
      steps {
        // Copy the resulting artifacts into common directory
        sh """
          ls target/*
          rm -rf oc-build && mkdir -p oc-build/deployments
          for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
            cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
          done
        """

        // Build container image using local Openshift cluster
        // Giving all the artifacts to OpenShift Binary Build
        // This places your artifacts into right location inside your S2I image
        // if the S2I image supports it.
        echo "projectName ${env.BUILD} and app_name ${env.APP_NAME}"
        //binaryBuild(projectName: env.BUILD, buildConfigName: env.APP_NAME, buildFromPath: "oc-build")
        binaryBuild(projectName: "basic-spring-boot-dev", buildConfigName: "basic-spring-boot-dev/simple-spring", buildFromPath: "oc-build")
      }
    }
  }
}
