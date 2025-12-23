pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_CLI_ARGS = '-no-color'
    }

    stages {
        stage('Load Credentials') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY'),
                    sshUserPrivateKey(credentialsId: 'SSH_CRED_ID', keyFileVariable: 'SSH_KEY')
                ]) {
                    echo "AWS and SSH credentials loaded securely."
                }
            }
        }

        stage('Terraform Initialization') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Variable Verification') {
            steps {
                script {
                    def varFile = "${env.BRANCH_NAME}.tfvars"
                    sh """
                        if [ -f ${varFile} ]; then
                            echo 'Variables in ${varFile}:'
                            cat ${varFile}
                        else
                            echo 'Variable file ${varFile} not found!'
                        fi
                    """
                }
            }
        }

        stage('Terraform Plan') {
    steps {
        script {
            def branchName = env.GIT_BRANCH ? env.GIT_BRANCH.replace('origin/', '') : 'dev'
            
            echo "Generating execution plan for branch: ${branchName}"

            sh "terraform plan -var-file=${branchName}.tfvars -out=tfplan"
        }
    }
}

        stage('Manual Approval') {
            steps {
                input message: "Approve Terraform Apply for branch ${env.BRANCH_NAME}?", ok: "Apply"
            }
        }

        stage('Terraform Apply') {
            steps {
                sh "terraform apply -auto-approve tfplan"
            }
        }
    }

    post {
        success {
            echo "Terraform deployment completed successfully."
        }
        failure {
            echo "Terraform deployment failed. Please check the logs."
        }
    }
}
