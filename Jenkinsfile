pipeline {
    agent {
        kubernetes {
            defaultContainer "maven"
            yamlFile "jenkins.k8s.yaml"
        }
    }
    options { 
        skipDefaultCheckout() 
        buildDiscarder(logRotator(numToKeepStr: "10"))
        disableConcurrentBuilds()
    }
    environment {
        ARTIFACTORY_SERVER = "jfrog"
        ARTIFACTORY = "sergeydzyuban.jfrog.io"
        DOCKER_REGISTRY = "docker-dev"
        DOCKER_LIVE_REGISTRY = "docker-live"
        APP_NAME = "jfrog/spring-petclinic"
    }
    stages {
        stage("Checkout") {
            steps {
                container("gitversion") {
                    script {
                        checkoutSCM(scm)
                        version()
                    }
                }
            }
        }

        stage("Init"){
            steps {
                rtServer (
                    id: env.ARTIFACTORY_SERVER,
                    url: "https://${env.ARTIFACTORY}/artifactory",
                    credentialsId: "jfrog-user-password"
                )
                configureMavenSettings()
            }
        }
        
        stage("Build") {
            steps {
                sh """
                    mvn versions:set -DnewVersion=${env.version} -s settings.xml >> $WORKSPACE/mvn.log 2>&1
                    mvn clean package -DskipTests=true -s settings.xml >> $WORKSPACE/mvn.log 2>&1
                """
            }
        }

        stage("Test") {
            steps {
                sh "mvn test -s settings.xml >> $WORKSPACE/mvn.test.log 2>&1"
            }
            post {
                always {
                    junit "target/surefire-reports/*.xml" 
                }
            }
        }

        stage("Docker.Build") {
            steps {
                container("docker") {
                    sh "docker build -t ${env.ARTIFACTORY}/${env.DOCKER_REGISTRY}/${env.APP_NAME}:${env.version} --build-arg VERSION=${env.version} . >> $WORKSPACE/docker.build.log 2>&1"
                }
            }
        }
        
        stage("Pack") {
            parallel {
                stage("Sonar") {
                    environment {
                        scannerHome = tool "sonar"
                    }
                    steps {
                        withSonarQubeEnv(installationName: "sonar") {
                            sh "mvn sonar:sonar -Dsonar.organization=sergeydz -Dsonar.projectKey=SergeyDz_spring-petclinic -s settings.xml >> $WORKSPACE/sonar.log 2>&1"
                        }
                    }
                }
                

                stage("Push") {
                    steps {
                        container("docker") {
                            rtDockerPush(
                                serverId: env.ARTIFACTORY_SERVER,
                                image: "${env.ARTIFACTORY}/${env.DOCKER_REGISTRY}/${env.APP_NAME}:${env.version}",
                                targetRepo: env.DOCKER_REGISTRY
                            )
                        }
                    }
                }
            }
        }

        stage("XRay") {
            steps {
                warnError(message: "XRay Scan failed") {
                    xrayScan (
                        serverId: env.ARTIFACTORY_SERVER,
                        failBuild: false
                    )
                }
            }
        }

        stage("Promote") {
            steps {
                rtAddInteractivePromotion(
                    serverId: env.ARTIFACTORY_SERVER,
                    targetRepo: env.DOCKER_LIVE_REGISTRY,
                    displayName: 'Promote Docker Image',
                    comment: 'Looks good!',
                    sourceRepo: env.DOCKER_LIVE_REGISTRY,
                    status: 'Released',
                    includeDependencies: true,
                    failFast: true,
                    copy: true
                )
            }
        }

        stage("Meta") {
            steps {
                rtPublishBuildInfo (
                    serverId: env.ARTIFACTORY_SERVER
                )
            }
        }
    }
    post {
        always {
            archiveArtifacts(artifacts: "*.log", allowEmptyArchive: true)
        }
    }
}

def checkoutSCM(scm) {
    checkout([
        $class: "GitSCM",
        branches: scm.branches,
        doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
        extensions: [
            [$class: "CloneOption", reference: "", noTags: false, shallow: false, depth: 0],
            [$class: "RelativeTargetDirectory", relativeTargetDir: ""]
        ],
        userRemoteConfigs: scm.userRemoteConfigs
    ])
}

def version() {
    def gitversion = sh(script: "/tools/dotnet-gitversion", returnStdout: true)
    env.version = readJSON(text: gitversion)?.FullSemVer

    sh "git config --global --add safe.directory ${workspace}"
    def author = sh(script: """git show -s --format=\"%ae\"""", returnStdout: true)

    currentBuild.description = "${env.version} by ${author}"
}

def configureMavenSettings() {
    withCredentials([string(credentialsId: "jfrog-token", variable: "JFROG_TOKEN")]) {
        def settings = readFile file: "settings.xml"
        writeFile file: "settings.xml", text: settings.replace("JFROG_TOKEN", env.JFROG_TOKEN)
    }
}