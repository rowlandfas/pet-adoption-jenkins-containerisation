pipeline {
    agent any
    stages {
//        stage('Code Checkout') {
//            steps {
//                git branch: 'main',
//                credentialsId: 'git-cred',
//                url: 'https://github.com/CloudHight/usteam.git'
//            }
//        }
        stage('Code Analysis') {
            steps {
               withSonarQubeEnv('sonarqube') {
                  sh "mvn sonar:sonar"
               }
            }
        }
        stage("Quality Gate") {
            steps {
              timeout(time: 2, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
          stage('Dependency check') {
              steps {
                  withCredentials([string(credentialsId: 'nvd-key', variable: 'NVD_API_KEY')]) {
                      dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey $NVD_API_KEY", 
                          odcInstallation: 'DP-Check'
                  }
                  dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
              }
          }

//        stage('Checkov Scan') {
//            steps {
//                sh '''
//                    # Ensure correct PATH for Checkov command
//                    export PATH=$PATH:$(python3 -m site --user-base)/bin
//
//                    # Run Checkov scan on the repository
//                    checkov -d . --output cli || true
//                '''
//            }
//        }
        stage('Build Artifact') {
            steps {
                sh 'mvn -f pom.xml clean package -DskipTests -Dcheckstyle.skip'
            }
        }
        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
                classifier: '',
                file: 'target/spring-petclinic-2.4.2.war',
                type: 'war']],
                credentialsId: 'nexus-cred',
                groupId: 'Petclinic',
                nexusUrl: 'nexus1.selfdevops.space',
                nexusVersion: 'nexus3',
                protocol: 'https',
                repository: 'nexus-repo',
                version: '1.0'
            }
        }
        stage('Build docker image') {
            steps {
                sshagent (['ansible-key']) {
                      // Install Docker collection first                     
                      sh 'ssh -t -t ec2-user@13.37.238.201 -o strictHostKeyChecking=no "ansible-galaxy collection install community.docker"'
                      sh 'ssh -t -t ec2-user@13.37.238.201 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook /opt/docker/docker-image.yml"'
                  }
              }
        }                
        stage('Trivy image scan') {
            steps {
                sh "trivy image cloudhight/testapp > trivyfs.txt"
            }             
        }
        stage('Deploy to stage') {
            steps {
                sshagent (['ansible-key']) {
                    sh "ssh -t -t ec2-user@13.37.238.201 -o strictHostKeyChecking=no \"cd /etc/ansible && ansible-playbook /opt/docker/docker-container.yml\""
                    sh "ssh -t -t ec2-user@13.37.238.201 -o strictHostKeyChecking=no \"cd /etc/ansible && ansible-playbook /opt/docker/newrelic-container.yml\""
                }
            }
        }
        
        stage('check website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://stage1.selfdevops.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage1.selfdevops.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack-cred')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack-cred')
                    }
                }
            }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }


        stage('Deploy to Prod') {
            steps {
                sshagent (['ansible-key']) {
                    sh "ssh -t -t ec2-user@13.37.238.201 -o strictHostKeyChecking=no \"cd /etc/ansible && ansible-playbook /opt/docker/docker-container.yml\""
                    sh "ssh -t -t ec2-user@13.37.238.201 -o strictHostKeyChecking=no \"cd /etc/ansible && ansible-playbook /opt/docker/newrelic-container.yml\""
                }
            }
        }

    }

}
