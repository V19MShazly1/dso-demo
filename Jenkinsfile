pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }

  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }

    stage('Static Analysis') {
      parallel {
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              sh '''
                #!/bin/bash 
                # Install dependencies
                apt-get update && apt-get install -y curl gpg gnupg2
                gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 105BD0E739499BDB
                # Install RVM
                 curl -sSL https://get.rvm.io | bash -s stable
                 echo 'source /usr/local/rvm/scripts/rvm' >> /etc/profile.d/rvm.sh                 
                 # Install Ruby (if necessary)
                 #rvm install 2.7.2
                #!/bin/bash --login
                #rvm use default
                gem install license_finder
                license_finder
              '''
            }
          }
        }
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
        }
      }
    }

    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        // TODO
        sh "echo done"
      }
    }
  }

  post {
    always {
      archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
      // dependencyCheckPublisher pattern: 'report.xml'
    }
  }
}
