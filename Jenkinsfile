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

                # Install Ruby and License Finder
                apt-get update && apt-get install -y ruby ruby-dev build-essential
                gem install license_finder

                # Run License Finder
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

        stage('Generate SBOM') {
          steps {
            container('maven') {
              sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
            }
          }
          post {
            success {
              dependencyTrackPublisher(
                projectName: 'sample-spring-app',
                projectVersion: '0.0.1',
                artifact: 'target/bom.xml',
                autoCreateProjects: true,
                synchronous: true
              )
              archiveArtifacts(
                allowEmptyArchive: true,
                artifacts: 'target/bom.xml',
                fingerprint: true,
                onlyIfSuccessful: true
              )
            }
          }
        }

        // ✅ New stage inserted here
        stage('Verify BOM') {
          steps {
            container('maven') {
              sh '''
                echo "[DEBUG] Checking if target/bom.xml exists..."
                ls -lh target/bom.xml || echo "⚠️ BOM file not found!"
              '''
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
        sh "echo done"
      }
    }
  }

  post {
    always {
      archiveArtifacts(
        allowEmptyArchive: true,
        artifacts: 'target/dependency-check-report.html',
        fingerprint: true,
        onlyIfSuccessful: true
      )
    }
  }
}
