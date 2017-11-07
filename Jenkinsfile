pipeline {
  agent {
    label "libvirt"
  }
  parameters {
    text(name: "COMMIT_MESSAGE", defaultValue: "", description: "Commit message to extract references from")
    string(name: "BRAIN_COMMIT", defaultValue: "master", description: "Brain project commit to checkout")
    string(name: "CELL_COMMIT", defaultValue: "master", description: "Cell project commit to checkout")
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
            BRAIN_PR = (COMMIT_MESSAGE =~ /Requires:.*\/brain\/pull\/(\d+)/)[0][1]
          } else {
            BRAIN_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/cell\/pull\/\d+/) {
            CELL_PR = (COMMIT_MESSAGE =~ /Requires:.*\/cell\/pull\/(\d+)/)[0][1]
          } else {
            CELL_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/hack\/pull\/\d+/) {
            HACK_PR = (COMMIT_MESSAGE =~ /Requires:.*\/hack\/pull\/(\d+)/)[0][1]
          } else {
            HACK_PR = null
          }
          if (COMMIT_MESSAGE =~ /Requires:.*\/cluster\/pull\/\d+/) {
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
          when { expression { HACK_PR } }
          steps {
            dir("hack") {
              sh("git fetch -f origin pull/${HACK_PR}/head:pull-request")
              sh("git checkout pull-request")
            }
          }
        }
        stage("cluster") {
          when { expression { CLUSTER_PR } }
          steps {
            dir("cluster") {
              sh("git fetch -f origin pull/${CLUSTER_PR}/head:pull-request")
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
