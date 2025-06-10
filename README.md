#  Ansible Project To Monitor VMs Health
<p>
Create EC2 instance 
  
Ansible-master – t2 medium with 20 gb

create 10 Workers – e2 small with 15 gb
</p>
### Step 1: Update the System

```
sudo apt update && sudo apt upgrade -y
```
### Step 2: Add the Ansible PPA

Ansible provides an official maintained PPA (for latest versions):

```
sudo add-apt-repository --yes --update ppa:ansible/ansible
```
### Step 3: Install Ansible

```
sudo apt install ansible –y
ansible --version
```
## Install AWS CLI
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```
## Tagging Script:
vi tag.sh
```
#!/bin/bash

# Fetch instance IDs that match Environment=dev and Role=web
instance_ids=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=dev" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

# Sort instance IDs deterministically
sorted_ids=($(echo "$instance_ids" | tr '\t' '\n' | sort))

# Rename instances sequentially
counter=1
for id in "${sorted_ids[@]}"; do
  name="web-$(printf "%02d" $counter)"
  echo "Tagging $id as $name"
  aws ec2 create-tags --resources "$id" \
    --tags Key=Name,Value="$name"
  ((counter++))
done

```
### create ssh key in ansible-master

```
ssh-keygen -t rsa -b 4096 -C "ansible-master"
```

### ansible.cfg (time to ssh conform yes , this conf check is disable)
vi ansible.cfg

```[defaults]
inventory = ./inventory/aws_ec2.yaml
host_key_checking = False

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```
### Dynamic Inventory

vi inventory/aws_ec2.yaml 

```
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  tag:Environment: dev
  instance-state-name: running
compose:
  ansible_host: public_ip_address
keyed_groups:
  - key: tags.Name
    prefix: name
  - key: tags.Environment
    prefix: env
```
```
# Step 1: Install venv module if not already present
sudo apt install python3-venv -y

# Step 2: Create a virtual environment
python3 -m venv ansible-env

# Step 3: Activate it
source ansible-env/bin/activate

# Step 4: Install required Python packages
pip install boto3 botocore docker
```
### ansible-galaxy collection install amazon.aws

```
ansible-inventory -i inventory/aws_ec2.yaml --graph
```
<p>
Copy Public Key in local  <nvkey.pem> and paste in new file nvkey.pem in ansible-master instance 
    
sudo chmod 400 nvkey.pem

</p>

### copy public key ansible-master to all workers 

vi copy-public-key.sh

```
#!/bin/bash

# Define vars
PEM_FILE="DevOps-Shack.pem"
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
USER="ubuntu"  # or ec2-user
INVENTORY_FILE="inventory/aws_ec2.yaml"

# Extract hostnames/IPs from dynamic inventory
HOSTS=$(ansible-inventory -i $INVENTORY_FILE --list | jq -r '._meta.hostvars | keys[]')

for HOST in $HOSTS; do
  echo "Injecting key into $HOST"
  ssh -o StrictHostKeyChecking=no -i $PEM_FILE $USER@$HOST "
    mkdir -p ~/.ssh && \
    echo \"$PUB_KEY\" >> ~/.ssh/authorized_keys && \
    chmod 700 ~/.ssh && \
    chmod 600 ~/.ssh/authorized_keys
  "
done

```
### check ssh is correct

```
ssh Ubuntu@ <ec2-3-86-193-164.compute-1.amazonaws.com> (copy)
```

# Create the Project 
create directory 
          
    mkdir vm-monitor

copy ansible.cfg and inventory into vm-monitor directory

mkdir /home/ubuntu/vm-monitor/group_vars

vi /home/ubuntu/vm-monitor/group_vars/all.yaml

```
smpt-server: “smtp.gmail.com”
smtp_port: 587
email_user: “yourname@gmail.com”
email_pass: “emailpassword” <create app password in google account https://myaccount.google.com/u/2/apppasswords >
alert_recipient: “other email address”
```

