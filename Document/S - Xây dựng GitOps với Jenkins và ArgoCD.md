```
def githubAccount = 'nguyenanhdongvn'

dockerBuildCommand = './'
def version = "v1.${BUILD_NUMBER}"

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'https://harbor.dongna.com'
        DOCKER_IMAGE_NAME = "demo-app/demo-app"
        DOCKER_IMAGE = "harbor.dongna.com/${DOCKER_IMAGE_NAME}"
    }

    stages {
        stage('Checkout project') {
          steps {
            git branch: appSourceBranch,
                credentialsId: githubAccount,
                url: appSourceRepo
          }
        }
        stage('Build And Push Docker Image') {
            steps {
                script {
                    sh "git reset --hard"
                    sh "git clean -f"
                                        app = docker.build(DOCKER_IMAGE_NAME, dockerBuildCommand)
                    docker.withRegistry( DOCKER_REGISTRY, dockerhubAccount ) {
                       app.push(version)
                    }

                    sh "docker rmi ${DOCKER_IMAGE_NAME} -f"
                    sh "docker rmi ${DOCKER_IMAGE}:${version} -f"
                }
            }
        }

        stage('Update value in helm-chart') {
            steps {
                                withCredentials([usernamePassword(credentialsId: 'nguyenanhdongvn', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                sh """#!/bin/bash
                                           [[ -d ${helmRepo} ]] && rm -r ${helmRepo}
                                           git config --global user.email "jenkins@gmail.com"
                                           git config --global user.name "Jenkins"
                                           git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/nguyenanhdongvn/app-helmchart.git --branch ${appConfigBranch}
                                           cd ${helmRepo}
                                           sed -i 's|  tag: .*|  tag: "${version}"|' ${helmValueFile}
                                           git add . ; git commit -m "Update to version ${version}";git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/nguyenanhdongvn/app-helmchart.git
                                           cd ..
                                           [[ -d ${helmRepo} ]] && rm -r ${helmRepo}
                                           """
                                }
            }
        }
    }
}
```
