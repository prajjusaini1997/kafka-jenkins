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
                        credentialsId: 'github-creds',
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
                echo "Waiting for EC2..."
                sleep 60
                '''
            }
        }

        stage('Checkout Ansible') {
            steps {
                dir('kafka-role') {
                    git(
                        branch: "${BRANCH}",
                        credentialsId: 'github-creds',
                        url: "${ANSIBLE_REPO}"
                    )
                }
            }
        }

        stage('Configure Bastion Proxy') {
            steps {
                dir('kafka-terraform') {
                    sh '''
                    BASTION_IP=$(terraform output -raw bastion_ip)

                    echo "Using Bastion IP: $BASTION_IP"

                    cat > ../kafka-role/ssh.cfg <<EOF
Host *
    ProxyJump ubuntu@$BASTION_IP
EOF
                    '''
                }
            }
        }

        stage('Verify Inventory') {
            steps {
                dir('kafka-role') {
                    sh '''
                    ansible-inventory -i inventories/aws_ec2.yml --graph
                    '''
                }
            }
        }

        stage('Ansible Ping') {
            steps {
                dir('kafka-role') {
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh '''
                        export ANSIBLE_SSH_ARGS="-F $(pwd)/ssh.cfg"

                        ansible all \
                        -i inventories/aws_ec2.yml \
                        -u ubuntu \
                        -m ping
                        '''
                    }
                }
            }
        }

        stage('Deploy Kafka') {
            steps {
                dir('kafka-role') {
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh '''
                        export ANSIBLE_SSH_ARGS="-F $(pwd)/ssh.cfg"

                        ansible-playbook \
                        -i inventories/aws_ec2.yml \
                        playbooks/kafka.yml
                        '''
                    }
                }
            }
        }

        stage('Verify Kafka') {
            steps {
                dir('kafka-role') {
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh '''
                        export ANSIBLE_SSH_ARGS="-F $(pwd)/ssh.cfg"

                        ansible all \
                        -i inventories/aws_ec2.yml \
                        -u ubuntu \
                        -m shell \
                        -a "systemctl status kafka --no-pager"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployment Successful"
        }

        failure {
            echo "Deployment Failed"
        }

        always {
            cleanWs()
        }
    }
}
