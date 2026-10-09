pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '880247664825'
        ECR_REPOSITORY = 'aws-devops-website'
        IMAGE_TAG = 'v1'
        DOCKER = 'C:\\Users\\ets2h\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        AWS = 'C:\\Program Files\\Amazon\\AWSCLIV2\\aws.exe'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/lokeshwaranc25/aws-devops-cicd-project.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Building the website project...'
                bat 'dir'
            }
        }

        stage('Test') {
            steps {
                script {
                    if (!fileExists('index.html')) {
                        error 'index.html not found!'
                    }
                }

                echo 'Website file exists. Test passed!'
            }
        }

        stage('Docker Build') {
            steps {
                bat '"%DOCKER%" build -t %ECR_REPOSITORY%:%IMAGE_TAG% .'
            }
        }

        stage('Docker Login to ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat '''
                        set AWS_DEFAULT_REGION=%AWS_REGION%
                        "%AWS%" ecr get-login-password --region %AWS_REGION% | "%DOCKER%" login --username AWS --password-stdin %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com
                    '''
                }
            }
        }

        stage('Tag Docker Image') {
            steps {
                bat '"%DOCKER%" tag %ECR_REPOSITORY%:%IMAGE_TAG% %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPOSITORY%:%IMAGE_TAG%'
            }
        }

        stage('Push to ECR') {
            steps {
                bat '"%DOCKER%" push %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPOSITORY%:%IMAGE_TAG%'
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Docker image pushed to Amazon ECR!'
        }

        failure {
            echo 'FAILED: Check the Jenkins Console Output.'
        }
    }
}