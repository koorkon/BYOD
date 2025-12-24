pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
    }

    stages {

        /* ================= TASK 1 ================= */
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

        /* ================= TASK 2 ================= */
        stage('Create Dynamic Inventory') {
            steps {
                sh '''
                  echo "[splunk]" > dynamic_inventory.ini
                  echo "${INSTANCE_IP} ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/aws.pem" >> dynamic_inventory.ini
                '''
            }
        }

        /* ================= TASK 3 ================= */
        stage('Wait for EC2 Health Check') {
            steps {
                sh '''
                  aws ec2 wait instance-status-ok \
                  --instance-ids ${INSTANCE_ID} \
                  --region ${AWS_REGION}
                '''
            }
        }

        /* ================= TASK 4 ================= */
        stage('Install Splunk') {
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/splunk.yml',
                    inventory: 'dynamic_inventory.ini',
                    credentialsId: 'ansible-ssh-key'
                )
            }
        }

        stage('Test Splunk') {
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/test-splunk.yml',
                    inventory: 'dynamic_inventory.ini',
                    credentialsId: 'ansible-ssh-key'
                )
            }
        }

        /* ================= TASK 5 ================= */
        stage('Validate Destroy') {
            steps {
                input message: 'Proceed with Terraform Destroy?'
            }
        }

        stage('Terraform Destroy') {
            steps {
                dir('terraform') {
                    sh 'terraform destroy -auto-approve -var-file=dev.tfvars'
                }
            }
        }
    }

    /* ================= POST ACTIONS ================= */
    post {
        always {
            sh 'rm -f dynamic_inventory.ini'
        }

        failure {
            dir('terraform') {
                sh 'terraform destroy -auto-approve -var-file=dev.tfvars'
            }
        }

        aborted {
            dir('terraform') {
                sh 'terraform destroy -auto-approve -var-file=dev.tfvars'
            }
        }
    }
}
