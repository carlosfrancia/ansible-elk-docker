## ELK stack in EC2 powered by Ansible and Docker

This is an Ansible playbook to provision a Dockerized ELK stack (ELasticsearch-Logstash-Kibana) in AWS EC2. This will build a number of single ELK systems on Docker containers using a docker image of each component.

The following components will be installed on each EC2 instance:

- Docker
- Elasticsearch (Docker container)
- Logstash (Docker container)
- Kibana (Docker container)
- Filebeat (Shell)

## Requirements

### Software and dependencies

In order to run this playbook the following software and dependencies are required:

- Ansible
- python >= 2.6
- boto
- boto3
- docker-py >= 1.7.0
- Docker API >= 1.20
- docker-compose >= 1.7.0
- PyYAML >= 3.11

### AWS

Ensure you have an AWS access key and secret already configured on the shell environment variables, or with the relevant iam role for ec2 instance creation allocated to the instance from which Ansible runs.

A Amazon Virtual Private Cloud (VPC) and a subnet ID.

## Getting started

### Running this playbook

The deploy_elk_stak.yml playbook has three Ansible tags:

 - deploy, used to only deploy instances  
ansible-playbook -i hosts -t deploy deploy_elk_stack.yml

 - elk, used to only build the elk-stack (please note you will need a static inventory for this, as the deploy role builds the dynamic inventory)  
ansible-playbook -i hosts -t elk deploy_elk_stack.yml

 - filebeat, used to only deploy filebeat (please note you will need a static inventory for this, as the deploy role builds the dynamic inventory)
ansible-playbook -i hosts -t filebeat deploy_elk_stack.yml

 - A combination of tags can be used:
ansible-playbook -i hosts deploy_elk_stack.yml --tags "elk,filebeat"

### Changing the number of instances

To change the number of instances required, either set the count variable in roles/aws/defaults/main.yml, or pass in ec2_count=$requirednumber of instances via the command line:

ansible-playbook -i ec2.py deploy-elk-stack.yml -e ec2_count=15

### Hosts

This playbook is configured to be installed in a provisioned EC2 instance in AWS. The new IP addresses provided by EC2 are added to the host group elk-server within the inventory file (hosts).

To run this playbook in a provided non EC2 server the following changes must be done:

 - Add the desired IP address in the inventory file for ELK server and Filebeat servers.
 - Change the deploy_elk_stack.yml file to stop attempting to connect using ec2-user
 - Update logstash_URL in roles/filebeat/defaults/main.yml 

### Ports

The following ports are used between the Docker containers (only Kibana port is open to the world):

* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana
* 5440: Logstash for Beat Input
* 5400: Filebeat to send Beats

## Ansible configuration

### Deploy EC2 instance task

This task runs AWS role and it does the following two actions:

- Create a new security group which the minimum required open ports (5602 for Kibana).
- Create the desired number of EC2 instances to deploy ELK stack in each of them.

Parameters

* ec2_count: Number of desired EC2 instances
* vpc_id: Amazon Virtual Private Cloud ID
* ec2_region: EC2 region
* ec2_instance_type: EC2 instance type (t2.medium or larger is recommended, a smaller type can only be used if the JVM setup is changed).
* ec2_image: EC2 AMI id (Redhat version required)
* subnet_id: Amazon Subnet ID

### Deploy ELK Server

This role runs three roles:

#### Users role

Creation of a SSH user named elk (Public key under roles/users/files/elk.key). 

In order to use this user, key must be updated with the desired one.

#### Docker role

Installs the following packages to install Docker in the new server:

Yum dependencies

* docker
* device-mapper-libs
* device-mapper-event-libs

Python dependencies:

* docker-pycreds
* dockerpty
* docker: 3.1.4
* docker-compose: 1.20.1

These two docker versions are required to avoid an Ansible bug at the time this playbook is written (https://github.com/ansible/awx/issues/1197).

#### Elk-docker role

Installs three container for Elasticsearch, Logstash and Kibana:

* Elasticsearch: docker.elastic.co/elasticsearch/elasticsearch:6.1.3
* Logstash: docker.elastic.co/logstash/logstash:6.2.2
* Kibana: docker.elastic.co/kibana/kibana:6.1.3

#### Filebeat

Installs Filebeat software in the ELK Server. 

## Using my new ELK environment

### Elasticsearch

Check status on the health of the cluster (from the ELK server as 9200 port is not open in AWS):

curl -XGET '[ELK-SERVER]:9200/_cluster/health?pretty'

### Logstash

Configuration path location is mapped to /etc/logstash/config/ and therefore there is no need to access to the Docker container.

A Logstash pipeline is configured under /ELK-Ansible/roles/elk-docker/templates. 

The pipeline is configured to receive new log messages using port 544.   

Logs are sent to the standard output (for debugging) and to Elasticsearch.

Logs are sent to the index "my_index_1" which is created with the first message sent. 

This pipeline can be modified as per requirements but this container needs to be restarted to pick up the new settings.

### Filebeat

Filebeat is not install in a Docker container to add flexibility for making changes in its configuration.

Filebeat is configured under /home/carlos/github/ELK-Ansible/roles/filebeat/templates/filebeat.yml.j2 in Ansible

Filebeat configuration is in /etc/filebeat/filebeat.yml once is installed in the server.

A prospector to send server logs has been configured:

	- type: log
	  enabled: true
	  paths:
		- /var/log/*.log
		
Which then sends all the existing and new logs to Logstash:

	output.logstash:
	 hosts: ["192.168.181.132:5044"]

Execute Filebeat with the command: 

	sudo filebeat
	 
### Kibana

Access to Kibana using http://[ELK-SERVER]:5601.

Under Management/Index Patterns the new index is found (my_index_1) and a new index pattern can be created. 

All the messages will be then available in the Discover section.
