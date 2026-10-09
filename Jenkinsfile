pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Jenkins is preparing the project.'
            }
        }

        stage('Build') {
            steps {
                echo 'Building the website project.'
                echo 'Build completed successfully!'
            }
        }

        stage('Test') {
            steps {
                echo 'Testing the website project.'
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully!'
        }

        failure {
            echo 'CI pipeline failed. Check the logs.'
        }
    }
}