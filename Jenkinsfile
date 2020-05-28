pipeline {
  agent {
    docker {
      image 'scssubstratee/substratee_dev:18.04-2.9.1-1.1.2'
      args '''
        -u root
        --privileged
      '''
    }
  }
  options {
    timeout(time: 2, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '14'))
  }
  stages {
    stage('Build') {
      steps {
        sh 'cargo build | tee build.log'
      }
    }
    stage('Test') {
      steps {
        echo 'Stage TEST'
        sh 'BUILD_DUMMY_WASM_BINARY=1 cargo test --all'
      }
    }
    // running clippy doesn't actually make sense here, as it's 99% upstream code.
    // however, right now it didn't take much to make it pass
    stage('Clippy') {
      steps {
        sh 'cargo clean'
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh 'cargo clippy 2>&1 | tee clippy.log'
        }
      }
    }
    // NEVER!!! run cargo fmt! This is 99% upstream code and we need easy-rebase!
/*    stage('Formatter') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          sh 'cargo fmt -- --check > ${WORKSPACE}/fmt.log'
        }
      }
    } */
    stage('Results') {
      steps {
        recordIssues(
          aggregatingResults: true,
          enabledForFailure: true,
          qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]],
          tools: [
              cargo(
                pattern: 'build.log',
                reportEncoding: 'UTF-8'
              ),
              groovyScript(
                parserId:'clippy-warnings',
                pattern: 'clippy.log',
                reportEncoding: 'UTF-8'
              ),
              groovyScript(
                parserId:'clippy-errors',
                pattern: 'clippy.log',
                reportEncoding: 'UTF-8'
              )
          ]
        )
//        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
//                  sh './ci/check_fmt_log.sh'
//        }
      }
    }
    stage('Archive build output') {
      steps {
        archiveArtifacts {
          pattern('*.log')
          pattern('target/release/substratee-node')
          onlyIfSuccessful()
        }
      }
    }
  }
  post {
    unsuccessful {
        emailext (
          subject: "Jenkins Build '${env.JOB_NAME} [${env.BUILD_NUMBER}]' is ${currentBuild.currentResult}",
          body: "${env.JOB_NAME} build ${env.BUILD_NUMBER} is ${currentBuild.currentResult}\n\nMore info at: ${env.BUILD_URL}",
          to: "${env.RECIPIENTS_SUBSTRATEE}"
        )
    }
    always {
      cleanWs()
    }
  }
}
