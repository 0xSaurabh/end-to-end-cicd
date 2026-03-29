pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "0xsaurabh/java-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        RELEASE_NAME = "java-app"
        DOCKER_HOST = "tcp://13.201.103.103:2375"
    }

    stages {

        stage('Clone Code') {
            steps {
                git 'https://github.com/0xSaurabh/Spring-Boot-Sample-Project.git'
            }
        }

        stage('Build JAR') {
            steps {
                withMaven(maven: 'maven') {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$IMAGE_TAG .'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh 'docker push $DOCKER_IMAGE:$IMAGE_TAG'
                }
            }
        }

        stage('Deploy via Helm') {
            steps {
                sh """
                helm upgrade --install $RELEASE_NAME ./java-app \
                --set image.repository=$DOCKER_IMAGE \
                --set image.tag=$IMAGE_TAG
                """
            }
        }
    }
}
