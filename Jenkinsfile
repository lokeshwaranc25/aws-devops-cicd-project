pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '880247664825'
        ECR_REPOSITORY = 'aws-devops-website'
        IMAGE_TAG = 'v1'

        EC2_INSTANCE_ID = 'i-0be007d342447ffa4'

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

        stage('Test') {
            steps {
                script {
                    if (!fileExists('index.html')) {
                        error 'index.html not found!'
                    }
                }

                echo 'Website test passed!'
            }
        }

        stage('Docker Build') {
            steps {
                bat '"%DOCKER%" build -t %ECR_REPOSITORY%:%IMAGE_TAG% .'
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
                    '''
                }
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

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        def commands = [
                            "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com",
                            "docker pull ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}",
                            "docker stop aws-devops-container || true",
                            "docker rm aws-devops-container || true",
                            "docker run -d --name aws-devops-container --restart unless-stopped -p 80:80 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}",
                            "curl -f http://localhost/"
                        ]

                        def commandJson = groovy.json.JsonOutput.toJson([
                            commands: commands
                        ])

                        writeFile(
                            file: 'ssm-parameters.json',
                            text: groovy.json.JsonOutput.toJson([
                                commands: commands
                            ])
                        )

                        bat """
                            set AWS_DEFAULT_REGION=%AWS_REGION%
                            "%AWS%" ssm send-command ^
                                --region %AWS_REGION% ^
                                --instance-ids %EC2_INSTANCE_ID% ^
                                --document-name AWS-RunShellScript ^
                                --parameters file://ssm-parameters.json ^
                                --comment "Deploy AWS DevOps website from Jenkins" ^
                                --output json > ssm-command.json
                        """

                        def result = readJSON file: 'ssm-command.json'
                        def commandId = result.Command.CommandId

                        echo "SSM deployment command submitted: ${commandId}"

                        sleep time: 10, unit: 'SECONDS'

                        def status = bat(
                            script: """
                                "%AWS%" ssm get-command-invocation ^
                                    --region %AWS_REGION% ^
                                    --command-id ${commandId} ^
                                    --instance-id %EC2_INSTANCE_ID% ^
                                    --query Status ^
                                    --output text > ssm-status.txt
                            """,
                            returnStatus: true
                        )

                        if (status != 0) {
                            error 'Unable to retrieve deployment status from SSM.'
                        }

                        def deploymentStatus = readFile('ssm-status.txt').trim()

                        if (deploymentStatus != 'Success') {
                            error "Deployment did not succeed. SSM status: ${deploymentStatus}. Check the AWS Systems Manager command output."
                        }

                        echo 'EC2 deployment completed successfully!'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Docker image built, pushed to ECR, and EC2 deployment completed!'
        }

        failure {
            echo 'FAILED: Check the Jenkins Console Output to identify the failed stage.'
        }
    }
}