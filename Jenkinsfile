pipeline {
  agent {
    label "libvirt"
  }
  stages {
    stage("Clone dependencies") {
      steps {
        dir("hack") {
          git url: "https://github.com/smartbox-io/hack.git"
        }
      }
    }
    stage("Build cluster") {
      steps {
        dir("hack") {
          sh("./hack --cells 1")
        }
      }
    }
    stage("Wait for cluster") {
      steps {
        dir("hack") {
          sh("./hack --wait")
        }
      }
    }
  }
  post {
    always {
      dir("hack") {
        sh("./hack --destroy-all")
      }
    }
  }
}
