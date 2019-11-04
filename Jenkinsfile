pipeline {
  agent { docker { image 'netfoundry/ziti-build-pipeline:latest'}}

  stages {
    stage("Setup") {
      steps {
        sh 'git submodule update --init --recursive'
        sh 'git submodule status --recursive'
        sh 'git fetch --verbose --tags'
        script {
          git_info = sh(returnStdout: true, script: 'git describe --long')
          committer = sh(returnStdout: true, script: 'git show -s --pretty=%ae')
        }
      }
    }
    stage('Tagging') {
      when { branch 'master'}
      steps {
        script {
           try {
           def zitiVer = readFile('version').trim()
           echo "$zitiVer"
           def (zitiMajor, zitiMinor, zitiPatch) = zitiVer.split(/\./).collect{ it.toInteger() }

           def tagVer = sh(returnStdout: true, script: 'git describe --long')
           def (tagMajor, tagMinor, tagPatch, ahead) = tagVer.split(/[\.-]/).take(4).collect { it.toInteger() }

           if ( zitiMajor > tagMajor ||
                (zitiMajor == tagMajor && zitiMinor > tagMinor ) ||
                (zitiMinor == tagMajor && zitiMinor == tagMinor && zitiPatch > tagPatch) )
           {
                new_tag = zitiVer
                echo "advancing tag($new_tag) based on 'version' file"
           } else {
               if (ahead == "0") {
                    echo "already has tag = ${tagMajor}.${tagMinor}.${tagPatch}"
                    new_tag = "${tagMajor}.${tagMinor}.${tagPatch}"
               }
               else {
                    new_tag = "${tagMajor}.${tagMinor}.${tagPatch.toInteger() + 1}"
                    echo "setting new tag = $new_tag"
                    sh "git tag -a ${new_tag} -m \'CI tag ${new_tag} \'"
               }
           }
           } catch (ex) {
               echo "$ex"
               currentBuild.status = 'FAILURE'
           }
        }
      }
    }
    stage('Tests') {
      steps {
        sh 'mkdir build-test'
        dir('build-test') {
            sh 'cmake ..'
            sh 'cmake --build . --target all_tests'
            sh 'ctest --no-compress-output -T Test'
        }
      }
      post {
        always {
          // Archive the CTest xml output
          archiveArtifacts (
            artifacts: 'build-test/Testing/**/*.xml',
            fingerprint: true
          )

          // Process the CTest xml output with the xUnit plugin
          xunit (
            testTimeMargin: '3000',
            thresholdMode: 1,
            thresholds: [
              skipped(failureThreshold: '0'),
              failed(failureThreshold: '0')
            ],
          tools: [CTest(
              pattern: 'build-test/Testing/**/*.xml',
              deleteOutputFiles: true,
              failIfNotNew: false,
              skipNoTestFiles: true,
              stopProcessingIfError: true
            )]
          )
        }
      }
    }
    stage('Build all platforms') {
      parallel {
        stage('Linux-x86_64') {
          steps {
            echo "building ${STAGE_NAME}"
            sh "mkdir -p build-${STAGE_NAME}"
            dir("build-${STAGE_NAME}") {
               sh 'cmake -DCMAKE_BUILD_TYPE=Debug ..'
               sh 'cmake --build . --target package --target publish'
            }
          }
        }
        stage('Linux-arm') {
          steps {
            echo "building ${STAGE_NAME}"
            sh "mkdir -p build-${STAGE_NAME}"
            dir("build-${STAGE_NAME}") {
               sh 'cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_TOOLCHAIN_FILE=../toolchains/${STAGE_NAME}.cmake ..'
               sh 'cmake --build . --target package --target publish'
            }
          }
        }
        stage('Windows-x86') {
          when { expression { false } }
          steps {
            echo "building ${STAGE_NAME}"
            sh "mkdir -p build-${STAGE_NAME}"
            dir("build-${STAGE_NAME}") {
               sh 'cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_TOOLCHAIN_FILE=../toolchains/${STAGE_NAME}.cmake ..'
               sh 'cmake --build . --target package --target publish'
            }
          }
        }
        stage('Windows-x86_64') {
          steps {
            echo "building ${STAGE_NAME}"
            sh "mkdir -p build-${STAGE_NAME}"
            dir("build-${STAGE_NAME}") {
               sh 'cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_TOOLCHAIN_FILE=../toolchains/${STAGE_NAME}.cmake ..'
               sh 'cmake --build . --target package --target publish'
            }
          }
        }
      }
    }
    stage('git push tag') {
      when { branch 'master' }
      steps {
        sh 'git push origin ${new_tag}'
      }
    }
    stage('Publish') {
      steps {
        sh "./make_publish_spec.sh"
        sh "cat publish.json"
        rtUpload (
          serverId: 'ziti-uploads',
          specPath: "./publish.json"
        )
      }
    }
  }
  post {
    always {
      script {
         slackSend channel: 'dev-notifications',
           message: "${RUN_DISPLAY_URL}\n"+
            "*Branch*:    ${BRANCH_NAME}\n"+
            "*Version*:   ${git_info}\n"+
            "*Committer*: ${committer}\n"+
            "*Status*:    ${currentBuild.currentResult}\n"+
            "*Duration*:  ${currentBuild.durationString}\n"
      }
    }
  }
}