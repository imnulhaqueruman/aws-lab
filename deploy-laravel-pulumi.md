# Deploy Laravel with MySQL on AWS using Pulumi (Python)

This guide demonstrates how to use Pulumi with Python to automate the deployment of a Laravel application with MySQL on AWS.

## Prerequisites
- Python 3.7 or later
- Pulumi CLI
- AWS CLI configured
- AWS account with appropriate permissions
- Docker and Docker Compose

## # Create a Makefile with the following content to setup aws cli , pulumi and venv envrionment in poridhi VS Code server 
```makefile
# Makefile for AWS CLI and Pulumi Installation and Configuration

.PHONY: all install-aws install-pulumi configure-aws configure-pulumi clean verify help

# Default target
all: install-aws install-pulumi configure-aws configure-pulumi verify

# Variables
AWS_CLI_VERSION := 2
PULUMI_VERSION := 3.91.1
AWS_REGION := ap-northeast-3
PULUMI_ACCESS_TOKEN ?= 
AWS_ACCESS_KEY_ID ?= 
AWS_SECRET_ACCESS_KEY ?= 

# Install AWS CLI
install-aws:
	@echo "Installing AWS CLI v$(AWS_CLI_VERSION)..."
	sudo apt-get update
	sudo apt-get install -y unzip curl
	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
	unzip awscliv2.zip
	sudo ./aws/install
	rm -rf aws awscliv2.zip
	@echo "AWS CLI installation completed."

# Install Pulumi
install-pulumi:
	@echo "Installing Pulumi v$(PULUMI_VERSION)..."
	curl -fsSL https://get.pulumi.com | sh
	export PATH=$$PATH:$$HOME/.pulumi/bin
	@echo "Pulumi installation completed."

# Configure AWS CLI
configure-aws:
	@echo "Configuring AWS CLI..."
	@if [ -z "$(AWS_ACCESS_KEY_ID)" ]; then \
		read -p "Enter AWS Access Key ID: " aws_access_key_id; \
		read -p "Enter AWS Secret Access Key: " aws_secret_access_key; \
		aws configure set aws_access_key_id "$$aws_access_key_id" --profile default; \
		aws configure set aws_secret_access_key "$$aws_secret_access_key" --profile default; \
	else \
		aws configure set aws_access_key_id "$(AWS_ACCESS_KEY_ID)" --profile default; \
		aws configure set aws_secret_access_key "$(AWS_SECRET_ACCESS_KEY)" --profile default; \
	fi
	aws configure set region $(AWS_REGION) --profile default
	aws configure set output json --profile default
	@echo "AWS CLI configuration completed."

# Configure Pulumi
configure-pulumi:
	@echo "Configuring Pulumi..."
	@if [ -z "$(PULUMI_ACCESS_TOKEN)" ]; then \
		read -p "Enter Pulumi Access Token: " pulumi_token; \
		pulumi login --local || pulumi login "$$pulumi_token"; \
	else \
		pulumi login --local || pulumi login "$(PULUMI_ACCESS_TOKEN)"; \
	fi
	@echo "Pulumi configuration completed."

# Install Python and required packages
install-python-deps:
	@echo "Installing Python dependencies..."
	sudo apt-get update
	sudo apt-get install -y python3 python3-pip python3-venv
	python3 -m pip install --upgrade pip
	python3 -m pip install pulumi pulumi-aws
	pip install cryptography
	@echo "Python dependencies installed."

# Create new Pulumi project
create-project:
	@echo "Creating new Pulumi project..."
	@read -p "Enter project name: " project_name; \
	read -p "Enter project description: " project_description; \
	mkdir -p $$project_name && cd $$project_name && \
	pulumi new aws-python --name $$project_name --description "$$project_description" --force

# Verify installations
verify:
	@echo "Verifying installations..."
	@echo "AWS CLI Version:"
	aws --version
	@echo "\nPulumi Version:"
	pulumi version
	@echo "\nAWS Configuration:"
	aws configure list
	@echo "\nAll verifications completed."

# Clean up
clean:
	@echo "Cleaning up installation files..."
	rm -f awscliv2.zip
	rm -rf aws
	@echo "Cleanup completed."

# Help
help:
	@echo "Available targets:"
	@echo "  all                  - Install and configure everything (default)"
	@echo "  install-aws         - Install AWS CLI"
	@echo "  install-pulumi      - Install Pulumi"
	@echo "  configure-aws       - Configure AWS CLI"
	@echo "  configure-pulumi    - Configure Pulumi"
	@echo "  install-python-deps - Install Python and required packages"
	@echo "  verify             - Verify installations"
	@echo "  clean              - Clean up installation files"
	@echo "  help               - Show this help message"
	@echo "\nEnvironment variables:"
	@echo "  AWS_ACCESS_KEY_ID      - AWS Access Key ID"
	@echo "  AWS_SECRET_ACCESS_KEY  - AWS Secret Access Key"
	@echo "  AWS_REGION            - AWS Region (default: us-east-1)"
	@echo "  PULUMI_ACCESS_TOKEN   - Pulumi Access Token"
```

## Usage Instructions

1. **Create the Makefile**:
```bash
nano Makefile
# Paste the content above
```

2. **Make the file executable**:
```bash
chmod +x Makefile
```

3. **Run all installations and configurations**:
```bash
make all
```

4. **Run specific targets**:
```bash
# Install only AWS CLI
make install-aws

# Install only Pulumi
make install-pulumi

# Configure AWS
make configure-aws

# Configure Pulumi
make configure-pulumi

# Install Python dependencies
make install-python-deps

# Verify installations
make verify
```

5. **Using environment variables**:
```bash
# Run with predefined credentials
AWS_ACCESS_KEY_ID=your_key \
AWS_SECRET_ACCESS_KEY=your_secret \
PULUMI_ACCESS_TOKEN=your_token \
make all
```

## Common Issues and Solutions

1. **Permission Issues**:
```bash
# If you get permission denied
sudo chown $USER:$USER Makefile
```

2. **Path Issues**:
```bash
# Add to ~/.bashrc
export PATH=$PATH:$HOME/.pulumi/bin
source ~/.bashrc
```

3. **Python Virtual Environment** (Optional):
```bash
# Add to Makefile before installing Python packages
python3 -m venv venv
source venv/bin/activate
```


## Project Structure

Your project structure should look like this:
```
pulumi-laravel-aws/
├── __main__.py
├── Pulumi.yaml
├── requirements.txt
└── venv/
```

## Infrastructure Code to create a VPC , Subnet , Security Group , Key Pair , EC2 Instance 

Replace the contents of `__main__.py` with:

```python:__main__.py
import pulumi
import pulumi_aws as aws
import os
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.backends import default_backend

# VPC Configuration
vpc = aws.ec2.Vpc(
    "main",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    enable_dns_support=True,
    tags={
        "Name": "laravel-vpc"
    }
)

# Internet Gateway
igw = aws.ec2.InternetGateway(
    "main",
    vpc_id=vpc.id,
    tags={
        "Name": "laravel-igw"
    }
)

# Public Subnet
public_subnet = aws.ec2.Subnet(
    "public",
    vpc_id=vpc.id,
    cidr_block="10.0.1.0/24",
    map_public_ip_on_launch=True,
    availability_zone="ap-northeast-3a",  # Change as needed
    tags={
        "Name": "laravel-public-subnet"
    }
)

# Route Table
route_table = aws.ec2.RouteTable(
    "main",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            gateway_id=igw.id,
        )
    ],
    tags={
        "Name": "laravel-route-table"
    }
)

# Route Table Association
route_table_association = aws.ec2.RouteTableAssociation(
    "main",
    subnet_id=public_subnet.id,
    route_table_id=route_table.id
)

# Security Groups
mysql_sg = aws.ec2.SecurityGroup(
    "mysql",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=3306,
            to_port=3306,
            cidr_blocks=["10.0.0.0/16"]
        ),
        # Add SSH access
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=22,
            to_port=22,
            cidr_blocks=["0.0.0.0/0"]
        )
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            protocol="-1",
            from_port=0,
            to_port=0,
            cidr_blocks=["0.0.0.0/0"]
        )
    ],
    tags={
        "Name": "mysql-sg"
    }
)

laravel_sg = aws.ec2.SecurityGroup(
    "laravel",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=22,
            to_port=22,
            cidr_blocks=["0.0.0.0/0"]
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=8000,
            to_port=8000,
            cidr_blocks=["0.0.0.0/0"]
        )
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            protocol="-1",
            from_port=0,
            to_port=0,
            cidr_blocks=["0.0.0.0/0"]
        )
    ],
    tags={
        "Name": "laravel-sg"
    }
)

# Generate SSH key pair
def generate_key_pair():
    # Generate private key
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048,
        backend=default_backend()
    )
    
    # Generate public key
    public_key = private_key.public_key().public_bytes(
        serialization.Encoding.OpenSSH,
        serialization.PublicFormat.OpenSSH
    ).decode('utf-8')
    
    # Generate private key in PEM format
    private_key_pem = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    ).decode('utf-8')
    
    return public_key, private_key_pem

# Generate the key pair
public_key, private_key_pem = generate_key_pair()

# Create the key pair in AWS
key_pair = aws.ec2.KeyPair("my-keypair",
    public_key=public_key,
    tags={
        "Name": "laravel-keypair"
    }
)

# Function to save private key and create SSH config
def save_key_and_create_config(args):
    mysql_ip, laravel_ip = args[0], args[1]
    
    # Save private key
    key_path = os.path.expanduser('~/.ssh/laravel_private_key.pem')
    with open(key_path, 'w') as f:
        f.write(private_key_pem)
    os.chmod(key_path, 0o400)
    
    # Create SSH config
    config = f"""# MySQL Instance
Host mysql-server
    HostName {mysql_ip}
    User ubuntu
    IdentityFile ~/.ssh/laravel_private_key.pem
    StrictHostKeyChecking no

# Laravel Instance
Host laravel-server
    HostName {laravel_ip}
    User ubuntu
    IdentityFile ~/.ssh/laravel_private_key.pem
    StrictHostKeyChecking no
"""
    
    config_path = os.path.expanduser('~/.ssh/config')
    # Append to existing config or create new
    with open(config_path, 'a') as f:
        f.write(config)
    
    return {
        'private_key_path': key_path,
        'ssh_config_path': config_path,
        'mysql_connection': f'ssh mysql-server',
        'laravel_connection': f'ssh laravel-server'
    }

# User Data Scripts
mysql_user_data = """#!/bin/bash
apt-get update
apt-get install -y mysql-server
systemctl start mysql
systemctl enable mysql
mysql -e "CREATE DATABASE poridhi;"
mysql -e "CREATE USER 'poridhi'@'%' IDENTIFIED BY 'Poridhi@123456';"
mysql -e "GRANT ALL PRIVILEGES ON poridhi.* TO 'poridhi'@'%';"
mysql -e "FLUSH PRIVILEGES;"
sed -i 's/bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl restart mysql
"""

laravel_user_data = """#!/bin/bash
apt-get update
apt-get install -y docker.io docker-compose
systemctl start docker
systemctl enable docker
"""

# EC2 Instances
mysql_instance = aws.ec2.Instance(
    "mysql",
    instance_type="t2.micro",
    ami="ami-05f4d8898209c4f55",
    subnet_id=public_subnet.id,
    vpc_security_group_ids=[mysql_sg.id],
    key_name=key_pair.key_name,
    user_data=mysql_user_data,
    tags={
        "Name": "mysql-server"
    }
)

laravel_instance = aws.ec2.Instance(
    "laravel",
    instance_type="t2.micro",
    ami="ami-05f4d8898209c4f55",
    subnet_id=public_subnet.id,
    vpc_security_group_ids=[laravel_sg.id],
    key_name=key_pair.key_name,
    user_data=laravel_user_data,
    tags={
        "Name": "laravel-server"
    }
)

# Export values and create SSH config
combined_output = pulumi.Output.all(
    mysql_instance.public_ip,
    laravel_instance.public_ip
).apply(save_key_and_create_config)

pulumi.export('mysql_public_ip', mysql_instance.public_ip)
pulumi.export('laravel_public_ip', laravel_instance.public_ip)
pulumi.export('ssh_config', combined_output)
```



## Deployment

1. **Preview Changes**:
```bash
pulumi preview
```

2. **Deploy Infrastructure**:
```bash
pulumi up
```

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/cf2b1a2d-e179-4ede-924d-d0db753f9797.png)




`After deployment you will get a ssh config file in your home directory ~/.ssh/config`



1. **Connect to Laravel instance**
```bash
ssh laravel-server
```
![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/8d8d9b15-3235-43d7-96d0-e8ab52d33148.png)


2. **Create docker-compose.yml file**

```bash
sudo nano docker-compose.yml
version: '3.8'

services:
  app:
    image: your-dockerhub-username/laravel-app:latest
    ports:
      - "8000:8000"
    environment:
      - APP_ENV=production
      - DB_HOST=${MYSQL_PRIVATE_IP}
      - DB_PORT=3306
      - DB_DATABASE=laravel_db
      - DB_USERNAME=laravel_user
      - DB_PASSWORD=laravel_password


4. **Start Laravel Application**:
```bash
docker-compose up -d
```
![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/8cb50c50-8c8f-4cbe-9037-2225d06e39de.png)





## Expose your Laravel application to the internet using the Poridhi Load Balancer  

1. Click to load balancer icon in Poridhi lab console, then open a prompt for you 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/7b748f81-2378-457b-86d2-4bfed94bb098.png)

`Paste your Laravel EC2 instance public IP and your container port 8000`

2. After creating load balancer you will get a URL for your Laravel app 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/dd20f3bd-b47a-43ec-89e0-0666dd0f0f3b.png)

Open the URL in your browser and you will see your Laravel app 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/bf93ad61-95f2-4c8a-b0bf-c6620420dba3.png)