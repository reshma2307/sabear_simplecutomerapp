pipeline {
    agent any
    tools {
        maven "MVN_HOME"
    }
    environment {
        // Nexus configurations
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "3.39.228.209:8081/"
        NEXUS_REPOSITORY = "sonarqube"
        NEXUS_CREDENTIAL_ID = "nexus_keygen"
        
        // SonarQube configurations
        SCANNER_HOME = tool 'sonar_scanner'
        
        // Tomcat server configurations
        TOMCAT_HOST = "43.203.215.226:8080/"  // Tomcat server host
        TOMCAT_PORT = "22"  // SSH port for Tomcat (default SSH port is 22)
        TOMCAT_DEPLOY_PATH = "/opt/apache-tomcat-9.0.97/webapps"  // Path to Tomcat webapps directory
        
        // Jenkins Credential for Tomcat server (used for both SSH username/password or private key)
        TOMCAT_CREDENTIALS = credentials('tomcat_server') // 'tomcat_server' is the Jenkins credential ID

        // Slack integration credentials
        SLACK_CREDENTIALS = credentials('slack-integration') // 'slack-integration' is the Jenkins credential ID for Slack
    }
    stages {
        stage("clone code") {
            steps {
                script {
                    // Clone the source code from the specified branch
                    git branch: 'feature-1.1', url: 'https://github.com/betawins/sabear_simplecutomerapp.git'
                }
            }
        }
        stage("mvn build") {
            steps {
                script {
                    // Run Maven build, ignoring test failures
                    sh 'mvn -Dmaven.test.failure.ignore=true clean install'
                }
            }
        }
        stage('SonarCloud') {
            steps {
                withSonarQubeEnv('sonarqube_server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectKey=Ncodeit \
                    -Dsonar.projectName=Ncodeit \
                    -Dsonar.projectVersion=2.0 \
                    -Dsonar.sources=/var/lib/jenkins/workspace/$JOB_NAME/src/ \
                    -Dsonar.binaries=target/classes/com/visualpathit/account/controller/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports \
                    -Dsonar.jacoco.reportPath=target/jacoco.exec \
                    -Dsonar.java.binaries=src/com/room/sample
                    '''
                }
            }
        }
        stage("publish to nexus") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml"
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    artifactPath = filesByGlob[0].path
                    artifactExists = fileExists artifactPath
                    if (artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: artifactPath,
                                 type: pom.packaging],
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: "pom.xml",
                                 type: "pom"]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }
        stage("Deploy to Tomcat") {
            steps {
                script {
                    // Extract SSH username and password (or key) from Jenkins credentials
                    def tomcatUser = TOMCAT_CREDENTIALS.username
                    def tomcatPassword = TOMCAT_CREDENTIALS.password // This will contain the password, if available

                    // Check if we need to use SSH key or password authentication
                    if (tomcatPassword) {
                        // If password is available in the Jenkins credentials, use sshpass for password authentication
                        sh """
                        sshpass -p '${tomcatPassword}' scp -o StrictHostKeyChecking=no target/*.war ${tomcatUser}@${TOMCAT_HOST}:${TOMCAT_DEPLOY_PATH}
                        sshpass -p '${tomcatPassword}' ssh -o StrictHostKeyChecking=no ${tomcatUser}@${TOMCAT_HOST} 'sudo systemctl restart tomcat'
                        """
                    } else {
                        // If SSH key is used, deploy using key-based authentication
                        sh """
                        scp -i /path/to/ssh/key target/*.war ${tomcatUser}@${TOMCAT_HOST}:${TOMCAT_DEPLOY_PATH}
                        ssh -i /path/to/ssh/key ${tomcatUser}@${TOMCAT_HOST} 'sudo systemctl restart tomcat'
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            // Send Slack notification for successful builds only
            slackSend(channel: '#jenkins-integration', 
                      color: 'good', 
                      message: "Build and deployment of project Ncodeit was successful!")
        }
        failure {
            echo "Build or deployment failed."
        }
    }
}
