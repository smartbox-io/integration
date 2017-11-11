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
          if (!params.COMMIT_MESSAGE) {
            COMMIT_MESSAGE = sh(returnStdout: true, script: "git rev-list --format=%B --max-count=1 ${GIT_COMMIT}").trim()
          } else {
            COMMIT_MESSAGE = params.COMMIT_MESSAGE
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/brain\/pull\/\d+/) {
            if (params.BRAIN_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a brain PR when forcing a specific commit"
              sh("exit 1")
            }
            BRAIN_PR = (COMMIT_MESSAGE =~ /Requires:.*\/brain\/pull\/(\d+)/)[0][1]
          } else {
            BRAIN_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/cell\/pull\/\d+/) {
            if (params.CELL_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a cell PR when forcing a specific commit"
              sh("exit 1")
            }
            CELL_PR = (COMMIT_MESSAGE =~ /Requires:.*\/cell\/pull\/(\d+)/)[0][1]
          } else {
            CELL_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/hack\/pull\/\d+/) {
            if (params.HACK_COMMIT) {
              echo "[FAILURE] Sorry, you cannot Require a hack PR when forcing a specific commit"
              sh("exit 1")
            }
            HACK_PR = (COMMIT_MESSAGE =~ /Requires:.*\/hack\/pull\/(\d+)/)[0][1]
          } else {
            HACK_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/cluster\/pull\/\d+/) {
            if (params.CLUSTER_COMMIT) {
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
          BRAIN_COMMIT = params.BRAIN_COMMIT
          CELL_COMMIT = params.CELL_COMMIT
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
          when { expression { HACK_PR || params.HACK_COMMIT } }
          steps {
            script {
              dir("hack") {
                if (HACK_PR) {
                  sh("git fetch -f origin pull/${HACK_PR}/head:pull-request")
                  sh("git checkout pull-request")
                } else if (params.HACK_COMMIT) {
                  sh("git checkout -fB integration ${params.HACK_COMMIT}")
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
          when { expression { CLUSTER_PR || params.CLUSTER_COMMIT } }
          steps {
            script {
              dir("cluster") {
                if (CLUSTER_PR) {
                  sh("git fetch -f origin pull/${CLUSTER_PR}/head:pull-request")
                  sh("git checkout pull-request")
                } else if (params.CLUSTER_COMMIT) {
                  sh("git checkout -fB integration ${params.CLUSTER_COMMIT}")
                }
              }
            }
          }
        }
      }
    }
    stage("Build kubernetes cluster") {
      steps {
        dir("hack") {
          sh("./hack --cells ${params.CELL_NUMBER}")
          sh("./hack --wait-for-cluster")
          sh("./hack --label-nodes")
        }
      }
    }
    stage("Prepare environment") {
      when { expression { BRAIN_COMMIT || CELL_COMMIT } }
      parallel {
        stage("Ensure images exist") {
          steps {
            script {
              if (BRAIN_COMMIT) {
                BRAIN_IMAGE_EXISTS = sh(returnStdout: true, script: "curl --write-out %{http_code} --silent --output /dev/null -I https://registry.smartbox.io/v2/smartbox/brain/manifests/${BRAIN_COMMIT}").trim() == "200"
                if (!BRAIN_IMAGE_EXISTS) {
                  build job: "brain/master", parameters: [
                    string(name: "BRAIN_COMMIT", value: BRAIN_COMMIT),
                    booleanParam(name: "SKIP_INTEGRATION", value: true)
                  ]
                }
              }
            }
            script {
              if (CELL_COMMIT) {
                CELL_IMAGE_EXISTS = sh(returnStdout: true, script: "curl --write-out %{http_code} --silent --output /dev/null -I https://registry.smartbox.io/v2/smartbox/cell/manifests/${CELL_COMMIT}").trim() == "200"
                if (!CELL_IMAGE_EXISTS) {
                  build job: "cell/master", parameters: [
                    string(name: "CELL_COMMIT", value: CELL_COMMIT),
                    booleanParam(name: "SKIP_INTEGRATION", value: true)
                  ]
                }
              }
            }
          }
        }
        stage("Patch cluster manifests") {
          steps {
            script {
              dir("cluster") {
                if (BRAIN_COMMIT) {
                  sh("find manifests -type f -name '*.yaml' | xargs sed -i 's#image: smartbox/brain#image: registry.smartbox.io/smartbox/brain:${BRAIN_COMMIT}#g'")
                }
                if (CELL_COMMIT) {
                  sh("find manifests -type f -name '*.yaml' | xargs sed -i 's#image: smartbox/cell#image: registry.smartbox.io/smartbox/cell:${CELL_COMMIT}#g'")
                }
              }
            }
          }
        }
      }
    }
    stage("Deploy smartbox.io") {
      steps {
        dir("hack") {
          sh("./hack --apply")
        }
      }
    }
    stage("Wait for cells") {
      steps {
        dir("hack") {
          sh("./hack --wait-for-cells")
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
