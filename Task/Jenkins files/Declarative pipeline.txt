pipeline {
    agent any
    tools {
        maven "MVN_HOME"
    }
    environment {
        // Nexus settings
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "44.206.236.146:8081/"
        NEXUS_REPOSITORY = "SimpleCustomerApp"
        NEXUS_CREDENTIAL_ID = "nexus_keygen"

        // SonarQube
        SCANNER_HOME = tool 'sonar_scanner'
    }

    stages {
        stage("clone code") {
            steps {
                git 'https://github.com/Shaik123-hu/sabear_simplecutomerapp.git'
            }
        }

        stage("mvn build") {
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true clean install'
            }
        }

        stage("SonarCloud") {
            steps {
                withSonarQubeEnv('sonarqube_server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Ncodeit \
                        -Dsonar.projectName=Ncodeit \
                        -Dsonar.projectVersion=2.0 \
                        -Dsonar.sources=$WORKSPACE/src/ \
                        -Dsonar.binaries=target/classes/com/visualpathit/account/controller/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports \
                        -Dsonar.java.binaries=src/com/room/sample
                    '''
                }
            }
        }

        stage("publish to nexus") {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"

                    def groupId    = pom.groupId
                    def artifactId = pom.artifactId
                    def version    = pom.version
                    def packaging  = pom.packaging

                    def filesByGlob = findFiles(glob: "target/*.${packaging}")
                    if (filesByGlob.length == 0) {
                        error "*** No artifact found in target/*.${packaging}"
                    }

                    def artifactPath = filesByGlob[0].path
                    if (fileExists(artifactPath)) {
                        echo "*** Uploading ${artifactPath} to Nexus (group: ${groupId}, version: ${version}, packaging: ${packaging})"

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: groupId,
                            version: version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: artifactId, classifier: '', file: artifactPath, type: packaging],
                                [artifactId: artifactId, classifier: '', file: "pom.xml", type: "pom"]
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
                withCredentials([usernamePassword(credentialsId: 'tomcat', usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASS')]) {
                    script {
                        // Find the WAR file built by Maven
                        def warFile = sh(script: "ls target/*.war | head -n 1", returnStdout: true).trim()
                        echo "Deploying ${warFile} to Tomcat at context path /SimpleCustomerApp..."
                        sh """
                            curl -u $TOMCAT_USER:$TOMCAT_PASS \
                                 -T ${warFile} \
                                 "http://52.23.219.234:8080/manager/text/deploy?path=/SimpleCustomerApp&update=true"
                        """
                    }
                }
            }
        }
    }

    // ✅ Post-build Slack Notifications
    post {
        success {
            slackSend(
                channel: "#jenkins-integration",
                color: "good",
                message: "✅ Build, Nexus Upload & Deployment Successful: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                tokenCredentialId: "Slackid"
            )
        }
        failure {
            slackSend(
                channel: "#jenkins-integration",
                color: "danger",
                message: "❌ Build/Deploy Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                tokenCredentialId: "Slackid"
            )
        }
    }
}
