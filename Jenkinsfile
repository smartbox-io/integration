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
          sh("./hack --debug --cells 1")
          sh("./hack --debug --wait")
          sh("./hack --label-nodes")
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