#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    // environment {
    //     DOCKER_REPO_SERVER = '330673547330.dkr.ecr.eu-central-1.amazonaws.com'
    //     DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
    // }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "docker build -t omarriad07:${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin ${DOCKER_REPO_SERVER}'
                        sh "docker push omarriad07:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('deploy') {
            environment{
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                   echo 'deploying docker image...'
                   sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                   sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update'){
            steps {
                script {
                    sshagent(['github-ssh-key']) {
                        sh "git remote set-url origin https://${USER}:${PASS}@gitlab.com/twn-devops-bootcamp/latest/11-eks/java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
    }
}
