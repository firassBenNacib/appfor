pipeline {
    agent any
    tools {
        maven 'Maven3'
    }
    environment {
        SCANNER_HOME = tool 'SonarScanner'
         APP_NAME = 'kube-keda' 
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('CheckOut') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/firassBenNacib/appfor']]])
            }
        }

        stage("Test Cases") {
            steps {
                sh "mvn clean test jacoco:report"
            }
        }

        stage("Package") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('Sonar-Server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=forapp-project \
                    -Dsonar.projectName='forapp-project' \
                    -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage("Quality gate") {
            steps {
                timeout(time: 4, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build docker image') {
            steps {
                script {
                      def imageName = "${APP_NAME}:${BUILD_NUMBER}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }

        stage('Push image to Hub') {
            steps {
                script {
                    def imageName = "${APP_NAME}:${BUILD_NUMBER}"
                    // Retrieve the Docker Hub password from the 'dockerhub-pwd' credential
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                        // Authenticate with Docker Hub
                        
                        sh "docker login -u firaskill12 -p ${dockerhubpwd}"



                      sh "docker tag ${imageName} your_dockerhub_username/${imageName}"
                    sh "docker push your_dockerhub_username/${imageName}"
                    }
                }
            }
        }

       stage('Update Helm Chart') {
            steps {
                script {
                    // Clone the Helm chart repository
                    dir('helm') {
                        git url: 'https://github.com/firassBenNacib/appfor-helm', branch: 'main'
                    }

                    // Update the values.yaml file with the new Docker image tag
                    sh "sed -i 's|imageTag: .*|imageTag: ${BUILD_NUMBER}|' helm/values.yaml"

                    // Commit and push the changes
                    git add 'helm/values.yaml'
                    git commit -m 'Update Docker image tag'
                    git push
                }
            }
        }
    }

    post {
        success {
            script {
                slackSend(
                    color: 'good',
                    message: "Build successful! :white_check_mark:",
                    channel: '#jenkins',
                    tokenCredentialId: 'Slack-Token'
                )
                emailext body: "Build successful!",
                         subject: "\$PROJECT_NAME - Build # \$BUILD_NUMBER - \$BUILD_STATUS!",
                         to: "firas.bennacib@esprit.tn",
                         mimeType: 'text/plain'
            }
        }
        failure {
            script {
                slackSend(
                    color: 'danger',
                    message: "Build failed! :x:",
                    channel: '#jenkins',
                    tokenCredentialId: 'Slack-Token'
                )
                emailext body: "Build failed!",
                         subject: "\$PROJECT_NAME - Build # \$BUILD_NUMBER - \$BUILD_STATUS!",
                         to: "firas.bennacib@esprit.tn",
                         mimeType: 'text/plain'
            }
        }
    }
}
