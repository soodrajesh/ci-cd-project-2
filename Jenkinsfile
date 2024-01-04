pipeline {
    agent any

    environment {
        DEV_AWS_ACCESS_KEY_ID = credentials('aws-dev-user')
        PROD_AWS_ACCESS_KEY_ID = credentials('aws-prod-user')
        DEV_AWS_REGION = 'us-west-2'
        PROD_AWS_REGION = 'us-west-2'
        DEV_TF_WORKSPACE = 'development'
        PROD_TF_WORKSPACE = 'production'
        SLACK_CHANNEL = 'jenkins-alerts'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }


        stage('Install Checkov') {
            steps {
                script {
                    sh "pip3 install checkov"
                    def checkovPath = sh(script: 'pip show checkov | grep "Location" | cut -d " " -f 2', returnStdout: true).trim()
                    env.PATH = "${checkovPath}:${env.PATH}"
                }
            }
        }

        stage('Terraform Init') {
            steps {
                script {
                    // Answer "yes" to the state migration prompt during init
                    sh 'echo "yes" | terraform init'
                }
            }
        }

        stage('Terraform Select Workspace') {
            steps {
                script {
                    def terraformWorkspace
                    def awsCredentialsId

                    if (env.BRANCH_NAME == 'main') {
                        terraformWorkspace = PROD_TF_WORKSPACE
                        awsCredentialsId = 'aws-prod-user'
                    } else {
                        terraformWorkspace = DEV_TF_WORKSPACE
                        awsCredentialsId = 'aws-dev-user'
                    }

                    def awsAccessKeyId

                    // Retrieve AWS credentials from Jenkins
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredentialsId, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        awsAccessKeyId = env.AWS_ACCESS_KEY_ID
                    }

                    echo "Using AWS credentials:"
                    echo "Credentials ID: ${awsCredentialsId}"

                    // Check if the Terraform workspace exists
                    def workspaceExists = sh(script: "terraform workspace list | grep -q ${terraformWorkspace}", returnStatus: true)

                    if (workspaceExists == 0) {
                        echo "Terraform workspace '${terraformWorkspace}' exists."
                    } else {
                        echo "Terraform workspace '${terraformWorkspace}' doesn't exist. Creating..."
                        sh "terraform workspace new ${terraformWorkspace}"
                    }

                    // Set the Terraform workspace
                    sh "terraform workspace select ${terraformWorkspace}"
                }
            }
        }


        stage('Checkov Scan') {
            steps {
                echo "Workspace Path: ${WORKSPACE}"
                echo "Running Checkov Scan"
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    // Run Checkov scan and capture the output, skipping tf.json
                    def checkovOutput = sh(script: 'checkov -d . --quiet --compact --skip-check tf.json', returnStdout: true).trim()

                    // Check for failed entries in the output
                    def failedChecks = checkovOutput.contains('FAILED for resource:')

                    // Print the output to the Jenkins console
                    echo "Checkov Scan Output:"
                    echo checkovOutput

                    // Throw an error if failedChecks is true
                    if (failedChecks) {
                        error 'Checkov scan found failed entries'
                    }
                }
            }
        }



        stage('Terraform Plan') {
            steps {
                script {
                    // Additional steps if needed
                    sh 'terraform plan -out=tfplan -lock=false'
                }
            }
        }
 
        stage('Manual Approval') {
            steps {
                script {
                    echo 'Waiting for approval...'
                    input message: 'Do you want to apply the Terraform plan?',
                          ok: 'Proceed'
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                script {
                    // Ensure awsCredentialsId is defined in this scope
                    def awsCredentialsId

                    if (env.BRANCH_NAME == 'main') {
                        awsCredentialsId = 'aws-prod-user'
                    } else {
                        awsCredentialsId = 'aws-dev-user'
                    }

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredentialsId, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh 'terraform apply -auto-approve tfplan'    
                    }

                    // Notify Slack about the successful apply
                    slackSend(
                        color: '#36a64f',
                        message: "Terraform apply successful on branch ${env.BRANCH_NAME}",
                        channel: SLACK_CHANNEL
                    )

                }
            }
        }
    }

post {
    always {
        // Notification for every build completion
        slackSend(
            color: '#36a64f',
            message: "Jenkins build ${env.JOB_NAME} ${env.BUILD_NUMBER} completed.\nPipeline URL: ${env.BUILD_URL}",
            channel: SLACK_CHANNEL
        )
        slackSend(
            color: '#36a64f',
            message: "GitHub build completed.\nPipeline URL: ${env.BUILD_URL}",
            channel: SLACK_CHANNEL
        )
    }

    failure {
        // Notification for build failure
        slackSend(
            color: '#FF0000',
            message: "Jenkins build ${env.JOB_NAME} ${env.BUILD_NUMBER} failed.\nPipeline URL: ${env.BUILD_URL}",
            channel: SLACK_CHANNEL
        )
    }

    unstable {
        // Notification for unstable build
        slackSend(
            color: '#FFA500',
            message: "Jenkins build ${env.JOB_NAME} ${env.BUILD_NUMBER} is unstable.\nPipeline URL: ${env.BUILD_URL}",
            channel: SLACK_CHANNEL
        )
    }

    aborted {
        // Notification for aborted build
        slackSend(
            color: '#FFFF00',
            message: "Jenkins build ${env.JOB_NAME} ${env.BUILD_NUMBER} aborted.\nPipeline URL: ${env.BUILD_URL}",
            channel: SLACK_CHANNEL
        )
    }
  }
}