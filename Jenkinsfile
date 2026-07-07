pipeline {
    agent any

    environment {
        TERRAFORM_REPO = 'https://github.com/prajjusaini1997/kafka-terraform.git'
        ANSIBLE_REPO   = 'https://github.com/prajjusaini1997/kafka-role.git'
        BRANCH         = 'main'
    }

        stages {

            stage('Checkout Terraform') {
                steps {
                    dir('kafka-terraform') {
                        git(
                           branch: "${BRANCH}",
                           credentialsId: "github-creds",
                           url: "${TERRAFORM_REPO}"
                        )
                    }
                 }
            }

        stage('Terraform Init') {
            steps {
                dir('kafka-terraform') {
                    sh 'terraform init -reconfigure'
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                dir('kafka-terraform') {
                    sh 'terraform validate'
                }
            }
        } 
 
        stage('Terraform Plan') {
            steps {
               dir('kafka-terraform') {
                   sh 'terraform plan'
               }
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('kafka-terraform') {
                    sh 'terraform apply -auto-approve'
                } 
            }
        }

        stage('Wait for EC2') {
            steps {
                sh '''
                 echo "Waiting for EC2 instances to become ready..."
                sleep 60
                '''
            }
        }

        stage('Checkout Ansible') {
            steps {
                dir('kafka-role') {
                    git(
                       branch: "${BRANCH}",
                       credentialsId: "github-creds",
                       url: "${ANSIBLE_REPO}"
                    )
                }
            }
        }

        stage('Verify Inventory') {
            steps {
                dir('kafka-role') {
                    sh '''
                    ansible-inventory --graph
                    '''
                }
            }
        }

        stage('Ansible Ping') {
            steps {
                dir('kafka-role') {
                    sh '''
                    ansible all -m ping
                    '''
                }
            }
        }

        stage('Deploy Kafka') {
            steps {
                dir('kafka-role') {
                    sh '''
                    ansible-playbook playbooks/kafka.yml
                    '''
                }
            }
        }

        stage('Verify Kafka') {
             steps {
                 dir('kafka-role') {
                     sh '''
                     echo "Kafka verification started..."

                     ansible all -m shell -a "systemctl status kafka --no-pager"

                     echo "Kafka verification completed"
                     '''
                 }
             }
        }
   }

        post {

        success {
            echo 'Deployment Successful'
        }

        failure {
            echo 'Deployment Failed'
        }
    }
}
