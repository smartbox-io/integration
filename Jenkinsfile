pipeline {
  agent {
    label "libvirt"
  }
  parameters {
    string(name: "INTEGRATION_BRANCH", defaultValue: "master", description: "Integration project branch to checkout")
    string(name: "HACK_BRANCH", defaultValue: "master", description: "Hack project branch to checkout")
    string(name: "CLUSTER_BRANCH", defaultValue: "master", description: "Cluster project branch to checkout")
    string(name: "BRAIN_BRANCH", defaultValue: "master", description: "Brain project branch to checkout")
    string(name: "CELL_BRANCH", defaultValue: "master", description: "Cell project branch to checkout")
    string(name: "CELL_NUMBER", defaultValue: "1", description: "Number of cells to deploy")
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
    stage("Checkout specific revisions") {
      parallel {
        stage("Hack") {
          when {
            expression { return params.HACK_BRANCH ==~ /^pull\/\d+/ }
          }
          steps {
            dir("hack") {
              sh("git fetch origin ${params.HACK_BRANCH}/head:pull-request")
              sh("git checkout pull-request")
            }
          }
        }
        stage("Cluster") {
          when {
            expression { return params.CLUSTER_BRANCH ==~ /^pull\/\d+/ }
          }
          steps {
            dir("cluster") {
              sh("git fetch origin ${params.CLUSTER_BRANCH}/head:pull-request")
              sh("git checkout pull-request")
            }
          }
        }
      }
    }
    stage("Build cluster") {
      steps {
        dir("hack") {
          sh("./hack --cells ${CELL_NUMBER}")
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
