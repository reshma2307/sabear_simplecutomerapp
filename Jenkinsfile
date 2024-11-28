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
                    // Extract SSH username and password using Jenkins credentials
                    withCredentials([usernamePassword(credentialsId: 'tomcat_server', usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASS')]) {
                        // Deploy the WAR file to Tomcat using password-based SSH authentication
                        sh """
                        sshpass -p '${TOMCAT_PASS}' scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_HOST}:${TOMCAT_DEPLOY_PATH}
                        sshpass -p '${TOMCAT_PASS}' ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} 'sudo systemctl restart tomcat'
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
