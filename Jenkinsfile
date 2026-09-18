pipeline {
    agent any

    environment {
        APP_NAME           = "Automated-CICD-App"
        RELEASE            = "1.0.0"
        DOCKER_USER        = "mohammed314"
        DOCKER_CRED_ID     = "docker-hub"
        IMAGE_NAME         = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG          = "${RELEASE}-${BUILD_NUMBER}"
        SONAR_HOST_URL     = "http://35.154.158.200:9000"
        JFROG_URL          = "http://43.204.111.118:8082/artifactory"
        NOTIFICATION_EMAIL = "muntajibadnan@gmail.com"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', url: 'https://github.com/muntajibadnan-svg/registration-app-cicd-project'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv('SonarQube-server') { 
                        sh "mvn sonar:sonar -Dsonar.host.url=${SONAR_HOST_URL}"
                    }
                }    
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false
                }    
            }
        }

        stage("Artifactory Configuration") {
            steps {
                rtServer (
                    id: "jfrog-server",
                    url: "${JFROG_URL}",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )      
            }
        }

        stage("Deploy Artifacts") {
            steps {
                rtMavenRun (
                    tool: "Maven",
                    pom: "pom.xml",
                    goals: "clean install",
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }

        stage("Publish Build Info") {
            steps {
                rtPublishBuildInfo (
                    serverId: "jfrog-server"
                )
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_CRED_ID) {
                        def docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }
    }

    post {
        success {
            script {
                try {
                    emailext (
                        body: '''<html>
                            <body>
                                <h2>Build Successful! ✅</h2>
                                <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                                <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                                <p><strong>Build Status:</strong> SUCCESS</p>
                                <p>Artifacts have been successfully deployed to JFrog Artifactory.</p>
                                <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                            </body>
                        </html>''', 
                        subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful ✅", 
                        mimeType: 'text/html',
                        to: "${NOTIFICATION_EMAIL}"
                    )
                } catch (Throwable e) {
                    echo "Email notification skipped (SMTP not configured or error): ${e.message}"
                }
            }
        }
        failure {
            script {
                try {
                    emailext (
                        body: '''${SCRIPT, template="groovy-html.template"}''', 
                        subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed ❌", 
                        mimeType: 'text/html',
                        to: "${NOTIFICATION_EMAIL}"
                    )
                } catch (Throwable e) {
                    echo "Email notification skipped (SMTP not configured or error): ${e.message}"
                }
            }
        }
    }
}
