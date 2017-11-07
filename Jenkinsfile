pipeline {
  agent {
    label "libvirt"
  }
  parameters {
    text(name: "COMMIT_MESSAGE", defaultValue: "", description: "Commit message to extract references from")
    string(name: "INTEGRATION_BRANCH", defaultValue: "master", description: "Integration project branch to checkout")
    string(name: "HACK_BRANCH", defaultValue: "master", description: "Hack project branch to checkout")
    string(name: "CLUSTER_BRANCH", defaultValue: "master", description: "Cluster project branch to checkout")
    string(name: "BRAIN_COMMIT", defaultValue: "master", description: "Brain project commit to checkout")
    string(name: "CELL_COMMIT", defaultValue: "master", description: "Cell project commit to checkout")
    string(name: "CELL_NUMBER", defaultValue: "1", description: "Number of cells to deploy")
  }
  stages {
    stage("Environment") {
      steps {
        sh("env")
      }
    }
    stage("Clone dependencies") {
      steps {
        dir("brain") {
          git url: "https://github.com/smartbox-io/brain.git"
        }
        dir("cell") {
          git url: "https://github.com/smartbox-io/cell.git"
        }
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
            expression { params.HACK_BRANCH != "master" }
          }
          steps {
            dir("hack") {
              sh("git fetch -f origin ${HACK_BRANCH}:${HACK_BRANCH}")
              sh("git checkout ${HACK_BRANCH}")
            }
          }
        }
        stage("Cluster") {
          when {
            expression { params.CLUSTER_BRANCH != "master" }
          }
          steps {
            dir("cluster") {
              sh("git fetch -f origin ${CLUSTER_BRANCH}:${CLUSTER_BRANCH}")
              sh("git checkout ${CLUSTER_BRANCH}")
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
