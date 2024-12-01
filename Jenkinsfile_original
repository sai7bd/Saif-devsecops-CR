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
    stage('Test') {
      parallel {
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
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
              // dependencyCheckPublisher pattern: 'report.xml'
                }
              }
            }
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
                    /bin/bash --login
                    rvm use default
                    gem install license_finder
                    license_finder
                    '''
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
        stage('OCI image build') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f "$(pwd)/Dockerfile-nonrootuser" -c "$(pwd)" --insecure --skip-tls-verify --cache=true --destination=docker.io/chandikas/dso-demo --verbosity=debug' 
              }
            }
          }
      }
    }
    stage('OCI Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/chandikas/dso-demo'
                }
            }
          }
        stage('Image Scan') {
          steps {
            container('docker-tools') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                  sh 'trivy image --exit-code 1 -format json --output trivy-report.json docker.io/chandikas/dso-demo'
                      }              
                  }
              }
          post {
            always {
              // Archive the Trivy report as an artifact
             archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
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
}
