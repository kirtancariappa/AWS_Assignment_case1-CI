pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        APP_NAME = "registerapp"
        RELEASE = "1.0.0"
        DOCKER_USER = "cary01"
    }


        stage("Checkout from SCM") {
            steps {
                git branch: 'main', url: 'https://github.com/kirtan8/register-app.git'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean install"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar'
                }
            }
        }

        stage("Docker Build & Push Image") {
            steps {
                script {
                    def imageTag = "${RELEASE}-${BUILD_NUMBER}"
                    def imageName = "${DOCKER_USER}/${APP_NAME}:${imageTag}"

                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh """
                            docker build -t ${imageName} .
                            docker push ${imageName}
                        """
                    }

                    // Set IMAGE_TAG as env variable for later use
                    env.IMAGE_TAG = imageTag
                }
            }
        }

       stage("Scan Image") {
          steps {
           echo "Scanning Image for Vulnerabilities"
           script {
            def imageTag = "${RELEASE}-${BUILD_NUMBER}"
            def imageName = "${DOCKER_USER}/${APP_NAME}:${imageTag}"

            // Clean up optional
            sh '''
                docker system prune -af || true
                rm -rf ~/.cache/trivy || true
            '''

            // Let it download DB first time
            sh "trivy image --scanners vuln ${imageName} > trivyresults.txt"
                }
           }
       }
        stage("Trigger CD Pipeline") {
            steps {
                script {
                    echo "Triggering downstream CD pipeline with IMAGE_TAG=${env.IMAGE_TAG}"
                    build job: 'gitops-register-app-cd',
                          wait: false,
                          parameters: [
                              string(name: 'IMAGE_TAG', value: env.IMAGE_TAG)
                          ]
                }
            }
        }
         post {
            failure {
                emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed", 
                      mimeType: 'text/html',to: "kscariappa25@gmail.com"
      }
            success {
                emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful", 
                     mimeType: 'text/html',to: "kscariappa25@gmail.com"
                    }      
               }
         }
    }
}
