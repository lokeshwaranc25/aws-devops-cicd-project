pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = '880247664825'
        ECR_REPOSITORY  = 'aws-devops-website'
        IMAGE_TAG       = 'v1'
        EC2_INSTANCE_ID = 'i-0be007d342447ffa4'

        ECR_URI = '880247664825.dkr.ecr.us-east-1.amazonaws.com/aws-devops-website'

        DOCKER = 'C:\\Users\\ets2h\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        AWS    = 'C:\\Program Files\\Amazon\\AWSCLIV2\\aws.exe'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo 'Source code checked out successfully.'
            }
        }

        stage('Test') {
            steps {
                script {
                    if (!fileExists('index.html')) {
                        error 'index.html is missing.'
                    }

                    if (!fileExists('Dockerfile')) {
                        error 'Dockerfile is missing.'
                    }

                    echo 'Basic project file checks passed.'
                }
            }
        }

        stage('Docker Build') {
            steps {
                bat '''
                    "%DOCKER%" build -t aws-devops-website:%IMAGE_TAG% .
                    if errorlevel 1 exit /b 1
                '''

                echo 'Docker image built successfully.'
            }
        }

        stage('AWS Credential Check') {
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

                        "%AWS%" sts get-caller-identity
                        if errorlevel 1 exit /b 1
                    '''
                }

                echo 'AWS credentials verified.'
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

                        "%AWS%" ecr get-login-password --region %AWS_REGION% | "%DOCKER%" login --username AWS --password-stdin %ECR_URI%
                        if errorlevel 1 exit /b 1
                    '''
                }

                echo 'Docker logged in to Amazon ECR.'
            }
        }

        stage('Tag Docker Image') {
            steps {
                bat '''
                    "%DOCKER%" tag aws-devops-website:%IMAGE_TAG% %ECR_URI%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''

                echo 'Docker image tagged for ECR.'
            }
        }

        stage('Push to ECR') {
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

                        "%DOCKER%" push %ECR_URI%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                    '''
                }

                echo 'Docker image pushed to ECR successfully.'
            }
        }

        stage('Prepare EC2 Deployment') {
            steps {
                script {
                    def commands = [
                        'set -e',

                        'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 880247664825.dkr.ecr.us-east-1.amazonaws.com',

                        'docker pull 880247664825.dkr.ecr.us-east-1.amazonaws.com/aws-devops-website:v1',

                        'docker stop aws-devops-container || true',

                        'docker rm aws-devops-container || true',

                        'docker run -d --name aws-devops-container --restart unless-stopped -p 80:80 880247664825.dkr.ecr.us-east-1.amazonaws.com/aws-devops-website:v1',

                        'sleep 5; for i in $(seq 1 12); do if curl --fail --silent --show-error http://127.0.0.1/ -o /dev/null; then echo "Website health check passed"; exit 0; fi; echo "Waiting for Nginx... attempt $i"; sleep 5; done; echo "Website health check failed"; exit 1'
                    ]

                    def parameters = groovy.json.JsonOutput.toJson([
                        commands: commands
                    ])

                    writeFile(
                        file: 'ssm-parameters.json',
                        text: parameters
                    )

                    echo 'EC2 deployment commands prepared.'
                }
            }
        }

        stage('Deploy to EC2') {
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

                        "%AWS%" ssm send-command ^
                            --region %AWS_REGION% ^
                            --instance-ids %EC2_INSTANCE_ID% ^
                            --document-name AWS-RunShellScript ^
                            --parameters file://ssm-parameters.json ^
                            --comment "Deploy AWS DevOps website from Jenkins" ^
                            --query "Command.CommandId" ^
                            --output text > ssm-command-id.txt

                        if errorlevel 1 exit /b 1
                    '''

                    script {
                        def commandId = readFile('ssm-command-id.txt').trim()

                        if (!commandId || commandId == 'None') {
                            error 'Failed to obtain the SSM command ID.'
                        }

                        echo "SSM command submitted: ${commandId}"

                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                def queryResult = bat(
                                    returnStatus: true,
                                    script: """
                                        set AWS_DEFAULT_REGION=%AWS_REGION%

                                        "%AWS%" ssm get-command-invocation ^
                                            --region %AWS_REGION% ^
                                            --command-id ${commandId} ^
                                            --instance-id %EC2_INSTANCE_ID% ^
                                            --query Status ^
                                            --output text > ssm-status.txt
                                    """
                                )

                                if (queryResult != 0) {
                                    echo 'Unable to retrieve SSM status yet. Retrying.'
                                    sleep time: 5, unit: 'SECONDS'
                                    return false
                                }

                                def deploymentStatus = readFile('ssm-status.txt').trim()

                                echo "Current deployment status: ${deploymentStatus}"

                                if (deploymentStatus == 'Success') {
                                    return true
                                }

                                if ([
                                    'Failed',
                                    'Cancelled',
                                    'TimedOut',
                                    'Cancelling'
                                ].contains(deploymentStatus)) {
                                    error "EC2 deployment failed. SSM status: ${deploymentStatus}. Command ID: ${commandId}"
                                }

                                sleep time: 5, unit: 'SECONDS'
                                return false
                            }
                        }

                        echo 'EC2 deployment completed successfully.'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: CI/CD pipeline completed. The website has been deployed to EC2.'
        }

        failure {
            echo 'FAILURE: The pipeline failed. Check the Jenkins Console Output and SSM command output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}