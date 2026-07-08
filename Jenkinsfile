pipeline {
    agent any

    options {
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION      = 'ap-south-1'

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

Host 172.31.*
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
  instance-state-name: running

compose:
  ansible_host: private_ip_address

groups:
  kafka: "'kafka' in tags.Role"

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
timeout=30

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

        stage('Verify Inventory') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
                    ansible-inventory --graph
                    ansible all -m ping
                    '''
                }
            }
        }

        stage('Deploy Kafka') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
                    ansible-playbook playbooks/kafka.yml
                    '''
                }
            }
        }

        stage('Kafka Validation') {
            steps {

                dir("${ANSIBLE_DIR}") {

                    sh '''
                    ansible tag_kafka -m shell -a "systemctl is-active zookeeper"
                    ansible tag_kafka -m shell -a "systemctl is-active kafka"

                    ansible tag_kafka -m shell -a "ss -lntp | grep 9092"

                    ansible tag_kafka -m shell -a "ss -lntp | grep 2181"
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

            cleanWs()
        }
    }
}
