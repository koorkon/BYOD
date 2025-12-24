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

        stage('Terraform Init & Verify') {
    steps {
        sh 'ls -la' 
        sh 'terraform init'
        script {
            echo "Verifying dev.tfvars..."
            sh "[ -f dev.tfvars ] && cat dev.tfvars || echo 'FILE NOT FOUND'"
        }
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
        dir('terraform') {
            sh '''
              terraform init
              terraform apply -auto-approve -var-file=dev.tfvars
            '''
            script {
                env.INSTANCE_IP = sh(
                    script: "terraform output -raw instance_public_ip",
                    returnStdout: true
                ).trim()

                env.INSTANCE_ID = sh(
                    script: "terraform output -raw instance_id",
                    returnStdout: true
                ).trim()
            }
        }
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
