pipeline {
    environment {
        ARGO_SERVER = '44.203.239.156:30531'
        AUTH_TOKEN = credentials('argocd-deployer-token')
    }
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
                /*
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
                            archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true
                        }
                    }
                }
                stage('OSS License Checker') {
                    steps {
                        container('licensefinder') {
                            sh '''
                                gem install license_finder
                                license_finder
                            '''
                        }
                    }
                }
                */
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
                            sh '/kaniko/executor -f Dockerfile -c . --insecure --skip-tls-verify --cache=true --destination=docker.io/sai7bd/dso-demo --verbosity=debug'
                        }
                    }
                }
            }
        }
        stage('Deploy to Dev') {
            steps {
                container('argocli') {
                    script {
                        try {
                            sh 'argocd app sync devsecops --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
                            sh 'argocd app wait devsecops --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
                        } catch (Exception e) {
                            error "ArgoCD sync or wait failed: ${e.getMessage()}"
                        }
                    }
                }
            }
        }
    }
}
