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
        dir("cluster") {
          git url: "https://github.com/smartbox-io/cluster.git"
        }
      }
    }
    stage("Build cluster") {
      steps {
        dir("hack") {
          sh("./hack --cells 1")
          sh("./hack --wait")
          sh("./hack --label-nodes")
        }
      }
    }
    stage("Deploy application") {
      steps {
        dir("hack") {
          sh("find ../cluster/manifests -type f -name '*.yaml' -not -name '*-template.yaml' | xargs cat | ./kubectl apply -f -")
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
