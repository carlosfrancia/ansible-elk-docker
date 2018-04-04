## Dockerized ELK stack

Used to provision an ephemeral lab for ELK training on ec2 with amazon linux. This will build a number of single ELK systems in Docker container. The following components will be installed on each ec2 instance:

- Docker
- Elasticsearch (Docker container)
- Logstash (Docker container)
- Kibana (Docker container)
- Filebeat (optional)

## Changing the number of instances

To change the number of instances required, either set the count variable in roles/deploy/defaults/main.yml, or pass in count=$requirednumber of instances via the command line:

ansible-playbook -i ec2.py deploy-elk-stack.yml -e count=15

## Pre-requisites

Ensure you have an aws access key and secret already configured on the shell environment variables, or with the relevant iam role for ec2 instance creation allocated to the instance from which ansible runs.

## Ansible Tags

The site.yml has three tags:

 - deploy, used to only deploy instances  
ansible-playbook -i ec2.py -t deploy deploy_elk_stack.yml

 - elk, used to only build the elk-stack (please note you will need a static inventory for this, as the deploy role builds the dynamic inventory)  
ansible-playbook -i hosts -t elk deploy_elk_stack.yml

 - filebeat, used to only deploy filebeat (static inventory is required)

## Using this repository

To use this repository:

- Prepare  virtualenv:  
pip install virtualenv  
virtualenv --system-site-packages ansible  
source ansible/bin/activate  

- Install ansible in virtualenv:  
pip install ansible

- Clone the github repo:  
git clone https://github.com/carlosfrancia/ELK-Ansible.git  
cd ELK-Ansible

- Set variables and vault for ansible usage
export ANSIBLE_CONFIG=~/zabbix-training-lab/ansible.cfg  
echo [VAULT] > ~/.vault

- Run ansible:
ansible-playbook -i ec2.py deploy_elk_stack.yml

## Usage

- Elasticsearch

Check status on the health of the cluster:

curl -XGET '[ELK-SERVER]:9200/_cluster/health?pretty'

- Kibana

URL: http://[ELK-SERVER]:5601

- Logstash

A Logstash pipeline is configured under /ELK-Ansible/roles/elk-docker/templates. 

These files can be modified to meet different requirements

- Filebeat

Filebeat is configured under /home/carlos/github/ELK-Ansible/roles/filebeat/templates/filebeat.yml.j2

A prospector to send server logs has been configured:

	- type: log
	  enabled: true
	  paths:
		- /var/log/*.log

