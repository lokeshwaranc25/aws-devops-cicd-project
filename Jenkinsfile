pipeline {
    agent any

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
                bat 'docker build -t aws-devops-website:v1 .'
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the logs.'
        }
    }
}