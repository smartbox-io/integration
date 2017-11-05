pipeline {
  agent {
    label "libvirt"
  }
  triggers { upstream(upstreamProjects: "brain, cell", threshold: hudson.model.Result.SUCCESS) }
  parameters {
    string(name: "HACK_COMMIT", defaultValue: "master", description: "Hack project commit to checkout")
    string(name: "CLUSTER_COMMIT", defaultValue: "master", description: "Cluster project commit to checkout")
    string(name: "BRAIN_COMMIT", defaultValue: "master", description: "Brain project commit to checkout")
    string(name: "CELL_COMMIT", defaultValue: "master", description: "Cell project commit to checkout")
  }
  stages {
    stage("Retrieve environment") {
      steps {
        sh("env")
      }
    }
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
            expression { return params.HACK_COMMIT ==~ /^pull\/\d+/ }
          }
          steps {
            dir("hack") {
              sh("git fetch origin ${params.HACK_COMMIT}/head:pull-request")
              sh("git checkout pull-request")
            }
          }
        }
        stage("Cluster") {
          when {
            expression { return params.CLUSTER_COMMIT ==~ /^pull\/\d+/ }
          }
          steps {
            dir("cluster") {
              sh("git fetch origin ${params.CLUSTER_COMMIT}/head:pull-request")
              sh("git checkout pull-request")
            }
          }
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

