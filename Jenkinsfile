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
        TOMCAT_HOST = "43.203.215.226"  // Tomcat server host (No port needed for HTTP deployment)
        TOMCAT_PORT = "8080"  // HTTP port for Tomcat (Tomcat Manager uses HTTP)
        TOMCAT_DEPLOY_PATH = "/opt/apache-tomcat-9.0.97/webapps"  // Path to Tomcat webapps directory
        TOMCAT_MANAGER_URL = "http://$TOMCAT_HOST:$TOMCAT_PORT/manager/text"  // Tomcat Manager URL
        
        // Jenkins Credential for Tomcat server (used for HTTP basic authentication)
        TOMCAT_CREDENTIALS = credentials('tomcat_server') // 'tomcat_server' is the Jenkins credential ID for HTTP deployment

        // Slack integration credentials
        SLACK_CREDENTIALS = credentials('slack-integration') // 'slack-integration' is the Jenkins credential ID for Slack
    }
    stages {
        stage("Clone Code") {
            steps {
                script {
                    // Clone the source code from the specified branch
                    git branch: 'feature-1.1', url: 'https://github.com/betawins/sabear_simplecutomerapp.git'
                }
            }
        }
        stage("Maven Build") {
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
                    -Dsonar.sources=src/ \
                    -Dsonar.binaries=target/classes
                    '''
                }
            }
        }
        stage("Publish to Nexus") {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"
                    def filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    def artifactPath = filesByGlob[0].path
                    def artifactExists = fileExists artifactPath
                    if (artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: '3.39.228.209:8081/',
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: 'sonarqube',
                            credentialsId: 'nexus_keygen',
                            artifacts: [
                                [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging],
                                [artifactId: pom.artifactId, classifier: '', file: "pom.xml", type: "pom"]
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
                withCredentials([usernamePassword(credentialsId: 'tomcat_server', usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASS')]) {
                    script {
                        // Define the WAR file and deployment URL for Tomcat Manager
                        def warFile = "target/*.war"
                        def deployUrl = "${TOMCAT_MANAGER_URL}/deploy?path=/yourAppName&update=true"

                        // Deploy the WAR file using Tomcat Manager App via HTTP basic authentication
                        sh """
                        curl --user $TOMCAT_USER:$TOMCAT_PASS --data-binary @$warFile $deployUrl
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
