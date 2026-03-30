pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO   = '208976686989.dkr.ecr.ap-south-1.amazonaws.com/my-ecr-repo'
        APP_NAME   = 'my-ecr-repo'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    dockerImage = docker.build("${ECR_REPO}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REPO
                '''
                sh "docker push ${ECR_REPO}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy with Helm') {
            steps {
                sh """
                    helm upgrade --install myapp ./mychart \
                    --namespace helm-deployment --create-namespace \
                    --set image.repository=${ECR_REPO} \
                    --set image.tag=${BUILD_NUMBER} \
                    --set service.type=LoadBalancer
                """
            }
        }
    }
}
