pipeline {
    agent any

    options {
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['DEPLOY', 'DESTROY'],
            description: 'Select Terraform action'
        )

        booleanParam(
            name: 'AUTO_APPROVE',
            defaultValue: true,
            description: 'Auto approve Terraform apply/destroy'
        )
    }

    environment {
        AWS_REGION     = 'us-east-1'
        TERRAFORM_REPO = 'https://github.com/prajjusaini1997/kafka-terraform.git'
        ANSIBLE_REPO   = 'https://github.com/prajjusaini1997/Kafka-role.git'
        BRANCH         = 'main'
        TF_DIR         = 'terraform'
        ANSIBLE_DIR    = 'ansible'
        SSH_KEY        = '/var/lib/jenkins/.ssh/ninja_key.pem'
    }

    stages {

        stage('1. Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('2. Clone Repos') {
            steps {
                script {
                    dir("${TF_DIR}") {
                        git branch: "${BRANCH}", url: "${TERRAFORM_REPO}"
                    }
                    dir("${ANSIBLE_DIR}") {
                        git branch: "${BRANCH}", url: "${ANSIBLE_REPO}"
                    }
                    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'PEM_FILE')]) {
                        sh '''
                            mkdir -p $(dirname "${SSH_KEY}")
                            cp "${PEM_FILE}" "${SSH_KEY}"
                            chmod 600 "${SSH_KEY}"
                        '''
                    }
                }
            }
        }

        stage('3. Terraform Init') {
            steps {
                dir("${TF_DIR}") {
                    sh 'terraform init -upgrade'
                }
            }
        }

        stage('4. Terraform Validate') {
            steps {
                dir("${TF_DIR}") {
                    sh 'terraform validate'
                }
            }
        }

        stage('5. Terraform Plan') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                        if [ "$ACTION" = "DESTROY" ]; then
                            terraform plan -destroy -out=tfplan
                        else
                            terraform plan -out=tfplan
                        fi
                    '''
                }
            }
        }

        stage('6. Approval') {
            when {
                expression { return params.AUTO_APPROVE == false }
            }
            steps {
                input message: "Approve Terraform ${params.ACTION}?", ok: 'Continue'
            }
        }

        stage('7. Terraform Apply or Destroy') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                        if [ "$ACTION" = "DESTROY" ]; then
                            if [ "$AUTO_APPROVE" = "true" ]; then
                                terraform destroy -auto-approve
                            else
                                terraform destroy
                            fi
                        else
                            if [ "$AUTO_APPROVE" = "true" ]; then
                                terraform apply -auto-approve tfplan
                            else
                                terraform apply tfplan
                            fi
                        fi
                    '''
                }
            }
        }

        stage('8. Generate Bastion SSH Config') {
            when {
                expression { return params.ACTION == 'DEPLOY' }
            }
            steps {
                dir("${TF_DIR}") {
                    sh '''
                        BASTION_IP=$(terraform output -raw bastion_ip)

                        mkdir -p ~/.ssh

                        cat > ~/.ssh/config <<EOF
Host bastion
    HostName ${BASTION_IP}
    User ubuntu
    IdentityFile ${SSH_KEY}
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host 10.0.*
    User ubuntu
    IdentityFile ${SSH_KEY}
    ProxyJump bastion
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
EOF

                        chmod 600 ~/.ssh/config
                    '''
                }
            }
        }

        stage('9. Configure Ansible') {
            when {
                expression { return params.ACTION == 'DEPLOY' }
            }
            steps {
                dir("${ANSIBLE_DIR}") {
                    sh '''
                        mkdir -p inventories

                        cat > inventories/aws_ec2.yml <<EOF
plugin: amazon.aws.aws_ec2

regions:
  - ${AWS_REGION}

hostnames:
  - private-ip-address

keyed_groups:
  - key: tags.Role
    prefix: tag

filters:
  tag:Role: kafka
  instance-state-name: running

compose:
  ansible_host: private_ip_address

strict: False
EOF

                        cat > ansible.cfg <<EOF
[defaults]
inventory=inventories/aws_ec2.yml
host_key_checking=False
remote_user=ubuntu
private_key_file=${SSH_KEY}
timeout=60
forks=1

[inventory]
enable_plugins=amazon.aws.aws_ec2

[ssh_connection]
ssh_args=-F ~/.ssh/config
pipelining=True
EOF
                    '''
                }
            }
        }

        stage('10. Kafka Deploy and Verify') {
            when {
                expression { return params.ACTION == 'DEPLOY' }
            }
            steps {
                dir("${ANSIBLE_DIR}") {
                    sh '''
                        echo "Waiting for EC2 instances to become SSH ready..."
                        until ansible tag_kafka -m ping >/dev/null 2>&1
                        do
                            echo "SSH not ready yet... retrying in 10 seconds"
                            sleep 10
                        done

                        echo "========== INVENTORY FILE =========="
                        cat inventories/aws_ec2.yml

                        echo "========== ANSIBLE CFG =========="
                        cat ansible.cfg

                        echo "========== INVENTORY LIST =========="
                        ansible-inventory --list

                        echo "========== INVENTORY GRAPH =========="
                        ansible-inventory --graph

                        echo "========== ANSIBLE PING =========="
                        ansible tag_kafka -m ping

                        echo "========== SSH CONNECTIVITY =========="
                        ansible tag_kafka -m shell -a 'hostname && hostname -I && uptime'

                        echo "========== DEPLOYING KAFKA =========="
                        ansible-playbook playbooks/kafka.yml --diff

                        echo "========== KAFKA BROKER VERIFICATION =========="
                        ansible tag_kafka -m shell -a '
                        grep broker.id /opt/kafka/config/server.properties
                        systemctl is-active kafka
                        ss -lntp | grep 9092
                        '

                        echo "========== KAFKA VALIDATION =========="
                        ansible tag_kafka -m shell -a '
                        hostname
                        systemctl is-active kafka
                        ss -lnt | grep :9092
                        if systemctl list-unit-files | grep -q "^zookeeper.service"; then
                            systemctl is-active zookeeper
                            ss -lnt | grep :2181
                        else
                            echo "ZooKeeper not installed on this broker."
                        fi
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                echo '========================================='
                echo "Terraform ${params.ACTION} completed successfully"
                if (params.ACTION == 'DEPLOY') {
                    echo 'Dynamic Bastion Configured'
                    echo 'AWS Dynamic Inventory Loaded'
                    echo 'Kafka Cluster Successfully Deployed'
                } else {
                    echo 'Infrastructure Destroyed Successfully'
                }
                echo '========================================='
            }
        }

        failure {
            echo '========================================='
            echo "Pipeline Failed during ${params.ACTION}"
            echo 'Check Terraform/Ansible Logs'
            echo '========================================='
        }

        always {
            echo "Workspace cleanup disabled for debugging"
            // cleanWs()
        }
    }
}
