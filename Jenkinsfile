pipeline {
  agent {
    label "libvirt"
  }
  parameters {
    text(name: "COMMIT_MESSAGE", defaultValue: "", description: "Commit message to extract references from")
    string(name: "HACK_COMMIT", defaultValue: "", description: "Hack project commit to checkout")
    string(name: "CLUSTER_COMMIT", defaultValue: "", description: "Cluster project commit to checkout")
    string(name: "BRAIN_COMMIT", defaultValue: "", description: "Brain project commit to checkout")
    string(name: "CELL_COMMIT", defaultValue: "", description: "Cell project commit to checkout")
    string(name: "CELL_NUMBER", defaultValue: "1", description: "Number of cells to deploy")
  }
  stages {
    stage("Retrieve build environment") {
      steps {
        script {
          if (!COMMIT_MESSAGE) {
            COMMIT_MESSAGE = sh(returnStdout: true, script: "git rev-list --format=%B --max-count=1 ${GIT_COMMIT}").trim()
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/brain\/pull\/\d+/) {
            if (BRAIN_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a brain PR when forcing a specific commit"
              sh("exit 1")
            }
            BRAIN_PR = (COMMIT_MESSAGE =~ /Requires:.*\/brain\/pull\/(\d+)/)[0][1]
          } else {
            BRAIN_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/cell\/pull\/\d+/) {
            if (CELL_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a cell PR when forcing a specific commit"
              sh("exit 1")
            }
            CELL_PR = (COMMIT_MESSAGE =~ /Requires:.*\/cell\/pull\/(\d+)/)[0][1]
          } else {
            CELL_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/hack\/pull\/\d+/) {
            if (HACK_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a hack PR when forcing a specific commit"
              sh("exit 1")
            }
            HACK_PR = (COMMIT_MESSAGE =~ /Requires:.*\/hack\/pull\/(\d+)/)[0][1]
          } else {
            HACK_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/cluster\/pull\/\d+/) {
            if (CLUSTER_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a cluster PR when forcing a specific commit"
              sh("exit 1")
            }
            CLUSTER_PR = (COMMIT_MESSAGE =~ /Requires:.*\/cluster\/pull\/(\d+)/)[0][1]
          } else {
            CLUSTER_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/integration\/pull\/\d+/) {
            INTEGRATION_PR = (COMMIT_MESSAGE =~ /Requires:.*\/integration\/pull\/(\d+)/)[0][1]
          } else {
            INTEGRATION_PR = null
          }
        }
      }
    }
    stage("Checkout dependencies") {
     parallel {
        stage("brain") {
          steps {
            dir("brain") {
              git url: "https://github.com/smartbox-io/brain.git"
            }
          }
        }
        stage("cell") {
          steps {
            dir("cell") {
              git url: "https://github.com/smartbox-io/cell.git"
            }
          }
        }
        stage("hack") {
          steps {
            dir("hack") {
              git url: "https://github.com/smartbox-io/hack.git"
            }
          }
        }
        stage("cluster") {
          steps {
            dir("cluster") {
              git url: "https://github.com/smartbox-io/cluster.git"
            }
          }
        }
      }
    }
    stage("Checkout requirements") {
      parallel {
        stage("brain") {
          when { expression { BRAIN_PR } }
          steps {
            dir("brain") {
              sh("git fetch -f origin pull/${BRAIN_PR}/head:pull-request-${BRAIN_PR}")
              script {
                BRAIN_COMMIT = sh("git rev-parse pull-request-${BRAIN_PR}")
              }
            }
          }
        }
        stage("cell") {
          when { expression { CELL_PR } }
          steps {
            dir("cell") {
              sh("git fetch -f origin pull/${CELL_PR}/head:pull-request-${CELL_PR}")
              script {
                CELL_COMMIT = sh("git rev-parse pull-request-${CELL_PR}")
              }
            }
          }
        }
        stage("hack") {
          when { expression { HACK_PR || HACK_COMMIT } }
          steps {
            script {
              dir("hack") {
                if (HACK_PR) {
                  sh("git fetch -f origin pull/${HACK_PR}/head:pull-request")
                  sh("git checkout pull-request")
                } else if (HACK_COMMIT) {
                  sh("git checkout -fb integration ${HACK_COMMIT}")
                }
              }
            }
          }
        }
        stage("integration") {
          when { expression { INTEGRATION_PR } }
          steps {
            script {
              sh("git fetch -f origin pull/${INTEGRATION_PR}/head:pull-request")
              sh("git checkout pull-request")
            }
          }
        }
        stage("cluster") {
          when { expression { CLUSTER_PR || CLUSTER_COMMIT } }
          steps {
            script {
              dir("cluster") {
                if (CLUSTER_PR) {
                  sh("git fetch -f origin pull/${CLUSTER_PR}/head:pull-request")
                  sh("git checkout pull-request")
                } else if (CLUSTER_COMMIT) {
                  sh("git checkout -fb integration ${CLUSTER_COMMIT}")
                }
              }
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
    stage("Patch cluster manifests") {
      when { expression { BRAIN_COMMIT || CELL_COMMIT } }
      steps {
        script {
          dir("cluster") {
            if (BRAIN_COMMIT) {
              sh("find manifests -type f -name '*.yaml' | xargs sed -i 's#image: smartbox/brain#image: registry.smartbox.io/smartbox/brain:${BRAIN_COMMIT}#'")
            }
            if (CELL_COMMIT) {
              sh("find manifests -type f -name '*.yaml' | xargs sed -i 's#image: smartbox/cell#image: registry.smartbox.io/smartbox/cell:${CELL_COMMIT}#'")
            }
          }
        }
      }
    }
    stage("Deploy") {
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
