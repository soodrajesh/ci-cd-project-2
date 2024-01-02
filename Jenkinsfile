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

        stage('Setup Python Virtual Environment') {
            steps {
                script {
                    sh 'pipenv install checkov'
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

        stage('Terraform Plan') {
            steps {
                script {
                    // Additional steps if needed
                    sh 'terraform plan -out=tfplan'
                }
            }
        }

        stage('Generate Terraform Plan JSON') {
            steps {
                script {
                    // Generate JSON representation of Terraform plan
                    sh 'terraform show -json tfplan > tf.json'
                }
            }
        }

        stage('Checkov Scan') {
            steps {
                script {
                    // Activate the virtual environment and run Checkov
                    sh 'source myenv/bin/activate && checkov -f tf.json'
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
