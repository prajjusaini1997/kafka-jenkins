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
            description: 'Auto approve Terraform apply'
        )
    }

    environment {
        AWS_REGION      = 'us-east-1'

        TERRAFORM_REPO  = 'https://github.com/prajjusaini1997/kafka-terraform.git'
        ANSIBLE_REPO    = 'https://github.com/prajjusaini1997/kafka-role.git'

        BRANCH          = 'main'

        TF_DIR          = 'terraform'
        ANSIBLE_DIR     = 'ansible'

        SSH_KEY         = '/var/lib/jenkins/.ssh/ninja_key.pem'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Clone Terraform Repo') {
            steps {
                dir("${TF_DIR}") {
                    git branch: "${BRANCH}",
                        url: "${TERRAFORM_REPO}"
                }
            }
        }

        stage('Clone Ansible Repo') {
            steps {
                dir("${ANSIBLE_DIR}") {
                    git branch: "${BRANCH}",
                        url: "${ANSIBLE_REPO}"
                }
            }
        }

        stage('Terraform Init') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                    terraform init -upgrade
                    '''
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                    terraform validate
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Approval') {
            steps {
                input message: 'Approve Terraform Apply?', ok: 'Apply'
            }
        }

        stage('Terraform Apply') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }

        stage('Generate Bastion SSH Config') {
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

        stage('Configure Dynamic Inventory') {
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
                    '''
                }
            }
        }

        stage('Configure ansible.cfg') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
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
ssh_args=-F /var/lib/jenkins/.ssh/config
pipelining=True
EOF
                    '''
                }
            }
        }

        stage('Wait for EC2 SSH') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
                    echo "Waiting for EC2 instances to become SSH ready..."

                    until ansible tag_kafka -m ping >/dev/null 2>&1
                    do
                        echo "SSH not ready yet... retrying in 10 seconds"
                        sleep 10
                    done

                    echo "All EC2 instances are SSH reachable."
                    '''
                }
            }
        }

        stage('Verify SSH Connectivity') {
            steps {
                dir("${ANSIBLE_DIR}") {
                    sh '''
                    echo "========== SSH CONNECTIVITY =========="

                    ansible tag_kafka -m shell -a ' 
                    echo "========== HOSTNAME =========="
                    hostname

                    echo "========== IP ADDRESS =========="
                    hostname -I
  
                    echo "========== UPTIME =========="
                    uptime
                    '
                    '''
                }
            }
        }

        stage('Verify Inventory') {
            steps {
                dir("${ANSIBLE_DIR}") {
                    sh '''
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
                    '''
                }
            }
        }

        stage('Deploy Kafka') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
                    echo "========== DEPLOYING KAFKA =========="
                    ansible-playbook playbooks/kafka.yml --diff
                    '''
                }
            }
        }
        
        stage('Kafka Broker Verification') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
                    ansible tag_kafka -m shell -a '
                    echo "===== Broker ID ====="
                    grep broker.id /opt/kafka/config/server.properties 

                    echo "===== Kafka Status ====="
                    systemctl is-active kafka 

                    echo "===== Port ====="
                    ss -lntp | grep 9092 
                    '
                    '''
                }
            }
        }

        stage('Kafka Validation') {
            steps {
                dir("${ANSIBLE_DIR}") {
                    sh '''
                    ansible tag_kafka -m shell -a '
                    echo "===== HOST ====="
                    hostname

                    echo "===== Kafka Status ====="
                    systemctl is-active kafka

                    echo "===== Kafka Port ====="
                    ss -lnt | grep :9092

                    if systemctl list-unit-files | grep -q "^zookeeper.service"; then
                        echo "===== ZooKeeper Status ====="
                        systemctl is-active zookeeper

                        echo "===== ZooKeeper Port ====="
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

            echo '========================================='
            echo 'Terraform Infrastructure Created'
            echo 'Dynamic Bastion Configured'
            echo 'AWS Dynamic Inventory Loaded'
            echo 'Kafka Cluster Successfully Deployed'
            echo '========================================='
        }

        failure {

            echo '========================================='
            echo 'Pipeline Failed'
            echo 'Check Terraform/Ansible Logs'
            echo '========================================='
        }

        always {
               echo "Workspace cleanup disabled for debugging"
               // cleanWs()
            
        }
    }
}
