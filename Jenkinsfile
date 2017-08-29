import java.text.SimpleDateFormat


def image_name = sprintf('xenial-heat-elements-%1$s.qcow2', [new SimpleDateFormat('yyyy-MM-dd').format(new Date())])
def server_ip = ''
def server_id = ''
def server_status = ''

def admin_password = 'jenkinstesting'
def key_name_test ='jenkinskey'
def network_name = 'jenkinsnet'
def flavor_name = 'jenkinsflavor'

def timeout_in_seconds = 1200
def container_name = 'heat_images/ubuntu/xenial'
def object_format = 'xenial-heat-agents-__DATE__.qcow2'
def recent_limit = 10
def today = new SimpleDateFormat('yyyy-MM-dd').format(new Date())

pipeline {
    agent any

    environment {
        SERVER_NAME = 'james-osa-heat-testing'
        SERVER_IP = ''
        SERVER_ID = ''
        SERVER_STATUS = ''

        // Flavor id of 7 is 15 GB Standard Instance
        FLAVOR_ID = 7

        // Image ID is for ubuntu 16.04
        IMAGE_ID = "f5046581-62ab-4078-8d4d-ce09e5de1e93"

        KEY_NAME = 'james-jenkins'
    }

    stages {

        stage('Cleanup') {
            steps {
                echo "Image name is ${image_name}"
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                dir('dib-utils') {
                    git branch: 'master', url: 'https://github.com/openstack/dib-utils'
                }

                dir('diskimage-builder') {
                    git branch: 'master', url: 'https://github.com/openstack/diskimage-builder.git'
                }

                dir('heat-agents') {
                    git branch: 'stable/ocata', url: 'https://github.com/openstack/heat-agents.git'
                }

                dir('tripleo-image-elements') {
                    git branch: 'stable/ocata', url: 'https://github.com/openstack/tripleo-image-elements.git'
                }

                dir('test_templates') {
                    git branch: 'master', url: 'https://github.com/absalon-james/test_templates.git'
                }

                dir('xenial-custom-elements') {
                    git branch: 'master', url: 'https://github.com/absalon-james/xenial-custom-elements.git'
                }

                sh """
                    # Fix for os-collect-config service
                     sed -i 's/WantedBy=multi-user.target/WantedBy=cloud-init.target/g' tripleo-image-elements/elements/os-collect-config/install.d/os-collect-config-source-install/10-os-collect-config
                """
            }
        }

        stage ('Build Image') {
            steps {
                sh """
                    mkdir -p ~/.cache/image-create
                    rm -rf ~/.cache/image-create

                    virtualenv venvs/imaging


                    BIN=venvs/imaging/bin
                    PYTHON=\${BIN}/python2.7
                    \${BIN}/pip install pyyaml
                    \${BIN}/pip install networkx
                    \${BIN}/pip install stevedore

                    \${BIN}/pip install diskimage-builder

                    # Where to find elements that do not come with diskimage-builder
                    export ELEMENTS_PATH="\${PWD}/tripleo-image-elements/elements:\${PWD}/heat-agents:\${PWD}/xenial-custom-elements/elements"

                    # Indicate what release
                    export DIB_RELEASE="xenial"

                    \${BIN}/disk-image-create \
                        -a amd64 \
                        vm \
                        ubuntu \
                        os-collect-config \
                        os-refresh-config \
                        os-apply-config \
                        heat-config \
                        heat-config-script \
                        heat-config-cfn-init \
                        python-dib-utils \
                        python-keystoneclient \
                        -o ${image_name}
                """
            }
        }

        stage('Provision OSA VM') {
            steps {
                sh """
                    set +x
                    . ~/openrc
                    set -x

                    python ~/lib/provision-aio.py \
                        --flavor-id ${env.FLAVOR_ID} \
                        --image-id ${env.IMAGE_ID} \
                        --key-name ${env.KEY_NAME} \
                        ${env.SERVER_NAME}
                """
                script {
                    server_ip = readFile('server-ip')
                    server_id = readFile('server-id')
                    server_status = readFile('server-status')
                }
            }
        }

        stage('Bootstrapping Ansible') {
            steps {
                sh """
                    |ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@${server_ip} <<-'ENDSSH'
                    |    set -e
                    |    BRANCH=stable/newton
                    |    apt-get update
                    |    git clone https://git.openstack.org/openstack/openstack-ansible /opt/openstack-ansible
                    |    cd /opt/openstack-ansible
                    |    git checkout \${BRANCH}
                    |    ./scripts/bootstrap-ansible.sh
                    |ENDSSH
                """.stripMargin()
            }
        }

        stage('Bootstrapping AIO') {
            steps {
                sh """
                    |ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@${server_ip} <<-'ENDSSH'
                    |    set -e
                    |    cd /opt/openstack-ansible
                    |    ./scripts/bootstrap-aio.sh
                    |ENDSSH
                """.stripMargin()
            }
        }

        stage ('Running Playbooks') {
            steps {
                sh """
                    |ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@${server_ip} <<-'ENDSSH'
                    |    set -e
                    |    rm /etc/openstack_deploy/conf.d/horizon.yml
                    |    rm /etc/openstack_deploy/conf.d/swift.yml

                    |    # Change admin password
                    |    sed -i 's/^keystone_auth_admin_password.*/keystone_auth_admin_password: ${admin_password}/' /etc/openstack_deploy/user_secrets.yml
                    |ENDSSH
                """.stripMargin()

                sh """
                    |ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@${server_ip} <<-'ENDSSH'
                    |    set -e
                    |    cd /opt/openstack-ansible
                    |    ./scripts/run-playbooks.sh
                    |ENDSSH

                    |ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@${server_ip} <<-'ENDSSH'
                    |set -e
                    |    cd /opt/openstack-ansible/playbooks
                    |    openstack-ansible os-tempest-install.yml
                    |ENDSSH
                """.stripMargin()
            }
        }

        stage('Setup Environment') {
            steps {
                sh """
                    export OS_USERNAME=admin
                    export OS_INTERFACE=publicURL
                    export OS_PASSWORD=${admin_password}
                    export OS_PROJECT_NAME=admin
                    export OS_TENANT_NAME=admin
                    export OS_PROJECT_DOMAIN_NAME=Default
                    export OS_USER_DOMAIN_NAME=Default
                    export OS_AUTH_URL=https://${server_ip}:5000/v3
                    export OS_IDENTITY_API_VERSION=3
                    export OS_AUTH_VERSION=3

                    openstack --insecure security group list -f json > security-groups.json
                    openstack --insecure project list -f json > projects.json
                    SECURITY_GROUP_ID=\$(python ~/lib/get-security-group.py projects.json security-groups.json)

                    openstack --insecure keypair create ${key_name_test} --public-key ~/.ssh/id_rsa.pub

                    echo "Security group id is \${SECURITY_GROUP_ID}"
                    openstack --insecure security group rule list \${SECURITY_GROUP_ID}
                    # Update security group to include ssh
                    openstack --insecure security group rule create \${SECURITY_GROUP_ID} --protocol tcp --dst-port 22

                    SUBNET_NAME=jenkinssubnet
                    ROUTER_NAME=jenkinsrouter

                    # Create network
                    openstack --insecure network create ${network_name}

                    # Create subnet
                    #openstack --insecure subnet create \${SUBNET_NAME} --network ${network_name} --subnet-range 10.10.10.0/24 --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4 # NoneType iteral bug
                    neutron --insecure subnet-create ${network_name} 10.10.10.0/24  --name \${SUBNET_NAME}  --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4

                    # Create router
                    #openstack --insecure router create \${ROUTER_NAME} #NoneType iterable bug
                    neutron --insecure router-create \${ROUTER_NAME}

                    # Set router gateway to public network
                    neutron --insecure router-gateway-set \${ROUTER_NAME} public

                    # add interface to subnet to router
                    #openstack --insecure router add subnet \${ROUTER_NAME} \${SUBNET_NAME} ##### Nonetype Iterable bug
                    neutron --insecure router-interface-add \${ROUTER_NAME} \${SUBNET_NAME}

                    # Upload test image to glance
                    openstack --insecure image create ${image_name} --container-format bare --disk-format qcow2 --file ${image_name}

                    # Create flavor
                    openstack --insecure flavor create ${flavor_name} --ram 512 --disk 20 --vcpus 1
                """
            }
        }

        stage ("Run heat template") {
            steps {
                sh """
                    export OS_USERNAME=admin
                    export OS_INTERFACE=publicURL
                    export OS_PASSWORD=${admin_password}
                    export OS_PROJECT_NAME=admin
                    export OS_TENANT_NAME=admin
                    export OS_PROJECT_DOMAIN_NAME=Default
                    export OS_USER_DOMAIN_NAME=Default
                    export OS_AUTH_URL=https://${server_ip}:5000/v3
                    export OS_IDENTITY_API_VERSION=3
                    export OS_AUTH_VERSION=3

                    openstack --insecure stack create \
                        --parameter keyname=${key_name_test} \
                        --parameter image=${image_name} \
                        --parameter flavor=${flavor_name} \
                        --parameter floating-network=public \
                        --parameter private-network=${network_name} \
                        --parameter orchestration_api_location=${server_ip} \
                        --template test_templates/test_software_config.yaml \
                        test_stack

                    python ~/lib/wait-for-stack.py test_stack ${timeout_in_seconds}

                """
            } // Steps

            post {
                always {
                    sh """
                        export OS_USERNAME=admin
                        export OS_INTERFACE=publicURL
                        export OS_PASSWORD=${admin_password}
                        export OS_PROJECT_NAME=admin
                        export OS_TENANT_NAME=admin
                        export OS_PROJECT_DOMAIN_NAME=Default
                        export OS_USER_DOMAIN_NAME=Default
                        export OS_AUTH_URL=https://${server_ip}:5000/v3
                        export OS_IDENTITY_API_VERSION=3
                        export OS_AUTH_VERSION=3

                        openstack --insecure stack delete --yes test_stack
                    """
                } // Always
            } // Post Stage
        } // Stage
    } // Stages

    post {
        always {
            echo "AIO server IP: ${server_ip}"
            echo "Image name: ${image_name}"
        }
        success {
            sh """
                set +x
                . ~/openrc
                set -x

                # Save good image to swift
                python ~/lib/archive-image.py ${container_name} ${object_format} ~/workspace/Images/ubuntu_16.04_heat_agents_pipeline/${image_name} --limit ${recent_limit} --date ${today}

                # Delete AIO
                nova delete ${server_id}
            """
            echo "Successfully tested image ${image_name}"
        }
    }
}
