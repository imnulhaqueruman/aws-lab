# Automated Laravel Deployment on AWS using Pulumi and GitHub Actions

This guide demonstrates how to automate the deployment of a Laravel application on AWS infrastructure using Pulumi for infrastructure as code and GitHub Actions for CI/CD. We'll set up a two-tier architecture with MySQL and Laravel running on separate EC2 instances.

## Architecture Overview

![Architecture Diagram]
- VPC with public subnet
- Two EC2 instances (MySQL and Laravel)
- Security Groups for both instances
- Automated key pair management
- GitHub Actions for CI/CD pipeline

## Prerequisites

- AWS Account with appropriate permissions
- GitHub Account
- Pulumi Account (optional - can use local backend)
- Docker Hub Account (for container registry)

## Project Structure
```
laravel-aws-deployment/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── infrastructure/
│   ├── __main__.py
│   └── Pulumi.yaml
├── laravel/
│   ├── Dockerfile
│   └── docker-compose.yml
└── Makefile
```

`After opening the Poridhi lab, click the VS Code icon` 

![mage](https://s3.brilliant.com.bd/blog-bucket/thumbnail/011d3bce-8e13-4fad-bc61-92df67d74c31.png)

`This will open a VS Code server instance for you ` 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/56de24f8-2f4f-4c85-b042-d9fc55f3c2ea.png)

### Create a Makefile to install Laravel prerequisites , clone the Laravel todo app repository and docker in Poridhi VS Code server . 

```makefile
.PHONY: all install-aws install-pulumi configure-aws configure-pulumi clean verify help setup-laravel

# Default target
all: install-dependencies setup-infrastructure setup-laravel

# Group 1: Dependencies Installation
install_dependencies: update install_aws install_pulumi install_python_deps install_php install_composer install_nodejs

# Group 2: Infrastructure Setup
setup_infrastructure: configure_aws configure_pulumi verify

# Variables
AWS_CLI_VERSION := 2
PULUMI_VERSION := 3.91.1
AWS_REGION := ap-northeast-3
PULUMI_ACCESS_TOKEN ?= 
AWS_ACCESS_KEY_ID ?= 
AWS_SECRET_ACCESS_KEY ?= 

# System Update
update:
	@echo "Updating system packages..."
	sudo apt update && sudo apt upgrade -y
	sudo apt install -y software-properties-common unzip curl
	@echo "System update completed."

# Install AWS CLI
install_aws:
	@echo "Installing AWS CLI v$(AWS_CLI_VERSION)..."
	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
	unzip awscliv2.zip
	sudo ./aws/install
	rm -rf aws awscliv2.zip
	@echo "AWS CLI installation completed."

# Install Pulumi
install_pulumi:
	@echo "Installing Pulumi v$(PULUMI_VERSION)..."
	curl -fsSL https://get.pulumi.com | sh
	echo 'export PATH=$$PATH:$$HOME/.pulumi/bin' >> ~/.bashrc
	source ~/.bashrc
	@echo "Pulumi installation completed."

# Step 2: Install PHP and Required Extensions
install_php:
	sudo apt install -y software-properties-common
	sudo add-apt-repository ppa:ondrej/php -y
	sudo apt update
	sudo apt install -y php8.1 php8.1-common php8.1-cli php8.1-curl php8.1-mbstring php8.1-xml php8.1-zip php8.1-mysql php8.1-gd unzip

# Step 3: Install Composer
install_composer:
	curl -sS https://getcomposer.org/installer | php
	sudo mv composer.phar /usr/local/bin/composer
	sudo chmod +x /usr/local/bin/composer

# Install Node.js
install_nodejs:
	@echo "Installing Node.js..."
	curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
	sudo apt install -y nodejs
	@echo "Node.js installation completed."

# Install Python Dependencies
install_python_deps:
	@echo "Installing Python dependencies..."
	sudo apt install -y python3 python3-pip python3-venv
	python3 -m pip install --upgrade pip
	python3 -m pip install pulumi pulumi-aws cryptography
	@echo "Python dependencies installed."

# Configure AWS CLI
configure_aws:
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
configure_pulumi:
	@echo "Configuring Pulumi..."
	@if [ -z "$(PULUMI_ACCESS_TOKEN)" ]; then \
		read -p "Enter Pulumi Access Token: " pulumi_token; \
		pulumi login --local || pulumi login "$$pulumi_token"; \
	else \
		pulumi login --local || pulumi login "$(PULUMI_ACCESS_TOKEN)"; \
	fi
	@echo "Pulumi configuration completed."

# Laravel Project Setup
setup_laravel: clone_repo setup_permissions setup_env install_assets

# Clone Laravel Repository
clone_repo:
	@echo "Cloning Laravel repository..."
	git clone https://github.com/imnulhaqueruman/laravel-to-do-app.git
	@echo "Repository cloned."

# Setup Laravel Permissions
setup_permissions:
	@echo "Setting up Laravel permissions..."
	cd laravel-to-do-app && \
	sudo chown -R $$USER:www-data storage && \
	sudo chown -R $$USER:www-data bootstrap/cache && \
	chmod -R 775 storage && \
	chmod -R 775 bootstrap/cache
	@echo "Permissions setup completed."

# Setup Laravel Environment
setup_env:
	@echo "Setting up Laravel environment..."
	cd laravel-to-do-app && \
	cp .env.example .env && \
	php artisan key:generate
	@echo "Environment setup completed."

# Install Laravel Assets
install_assets:
	@echo "Installing Laravel dependencies and assets..."
	cd laravel-to-do-app && \
	composer install && \
	npm install && \
	npm run dev
	@echo "Assets installation completed."

# Start Laravel Server
start_server:
	@echo "Starting Laravel server..."
	cd laravel-to-do-app && \
	php artisan serve

# Create Pulumi Project
create_project:
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
	@echo "\nPHP Version:"
	php -v
	@echo "\nComposer Version:"
	composer --version
	@echo "\nNode.js Version:"
	node --version
	@echo "\nNPM Version:"
	npm --version
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
	@echo "  install-dependencies - Install all required dependencies"
	@echo "  setup-infrastructure - Configure AWS and Pulumi"
	@echo "  setup-laravel       - Setup Laravel project"
	@echo "  install-aws         - Install AWS CLI"
	@echo "  install-pulumi      - Install Pulumi"
	@echo "  install-php         - Install PHP and extensions"
	@echo "  install-composer    - Install Composer"
	@echo "  install-nodejs      - Install Node.js and npm"
	@echo "  configure-aws       - Configure AWS CLI"
	@echo "  configure-pulumi    - Configure Pulumi"
	@echo "  start-server        - Start Laravel development server"
	@echo "  verify             - Verify all installations"
	@echo "  clean              - Clean up installation files"
	@echo "  help               - Show this help message"
```
## Laravel setup instructions

1. Install php 8.1 and extensions
```bash
make install_php
```
2. Install composer
```bash
make install_composer
```
3. Install node js
```bash
make install_nodejs
```
4. Laravel project setup
```bash
make setup_laravel
```
`After setup the laravel project update the .env file `


## Pulumi project setup instructions 

1. Install Pulumi
```bash
make install_pulumi
```

2. Configure Pulumi
```bash
make configure_pulumi
```

3. Create Pulumi project
```bash
make create_project
```
`create procject according to this image `

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/ce7ce0b9-6a1d-4572-8edb-6a139f63c9cb.png)

* Project name `pulumi_infra`
* Project description `Pulumi infrastructure for laravel`
* For dev stack press Enter 
* Press Enter for passphrase
* change region to `ap-northeast-3`

 
## AWS CLI setup instructions 

1. Install AWS CLI
```bash
make install_aws
```
2. Configure AWS CLI
```bash
make configure_aws
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
`The infrastructure ready to deploy`

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/db0c94da-16ec-423b-8647-03b9f833959d.png)

`Note: after deploy the infrastructure you should run the make start_server to migrate db and check connection`

3. Open a new terminal and create workflows in your github repository  folder 
```bash 
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
```
4. Copied the laravel_private_key.pem 
```bash 
cat /root/.ssh/laravel_private_key.pem
```
3. Set up GitHub Secrets:

`Go to your github repository and click setting -> secrets -> new repository secret`

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/68ab2245-a95f-4c14-be7e-52b6b9e303e6.png)

   - AWS_ACCESS_KEY_ID
   - AWS_SECRET_ACCESS_KEY
   - DOCKER_USERNAME
   - DOCKER_PASSWORD
   - SSH_PRIVATE_KEY

4. Paste this yaml file in .github/workflows/deploy.yml
```yaml
name: Deploy Laravel Application

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_IMAGE: poridhi/laravel:v4.3

  LARAVEL_HOST: 13.208.42.43  # Your Laravel instance IP

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      # Setup SSH key
      - name: Install SSH Key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/laravel_private_key.pem
          chmod 600 ~/.ssh/laravel_private_key.pem
          echo -e "Host laravel-server\n\tHostName ${{ env.LARAVEL_HOST }}\n\tUser ubuntu\n\tIdentityFile ~/.ssh/laravel_private_key.pem\n\tStrictHostKeyChecking no" > ~/.ssh/config

      # Build and push Docker image
      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: |
            ${{ env.DOCKER_IMAGE }}
          
      # Create docker-compose.yml
      - name: Create docker-compose.yml
        run: |
          cat > docker-compose.yml << 'EOL'
          version: '3.3'
          
          services:
            app:
              image: ${{ env.DOCKER_IMAGE }}
              ports:
                - "8000:8000"
              environment:
                - APP_ENV=production
                - DB_CONNECTION=mysql
                - DB_HOST=${{ secrets.MYSQL_HOST }}
                - DB_PORT=3306
                - DB_DATABASE=poridhi
                - DB_USERNAME=poridhi
                - DB_PASSWORD=Poridhi@123456
              restart: unless-stopped
          EOL

      
      # Deploy to EC2
      - name: Deploy to EC2
        run: |
          # Copy docker-compose.yml to server
          scp docker-compose.yml ubuntu@laravel-server:~/
          
          # Deploy on server
          ssh ubuntu@laravel-server << 'ENDSSH'
            # Add current user to docker group if not already added
            sudo usermod -aG docker $USER
            
            # Install Docker Compose if not installed
            if ! command -v docker-compose &> /dev/null; then
              sudo apt-get update
              sudo apt-get install -y docker-compose
            fi
            
            # Set correct permissions
            sudo chown $USER:docker docker-compose.yml
            
            # Stop existing containers
            sudo docker-compose down || true
            
            # Pull latest image
            sudo docker-compose pull
            
            # Start new containers
            sudo docker-compose up -d
            
            # Run migrations
            sudo docker-compose exec -T app php artisan migrate --force
            
            # Clear cache
            sudo docker-compose exec -T app php artisan cache:clear
            sudo docker-compose exec -T app php artisan config:clear
            sudo docker-compose exec -T app php artisan view:clear
          ENDSSH

      # Verify deployment
      - name: Verify deployment
        run: |
          # Wait for application to start
          sleep 30
          
          # Check if application is responding
          curl -s -o /dev/null -w "%{http_code}" http://${{ env.LARAVEL_HOST }}:8000/

      # Notify on success/failure
      - name: Notify on success
        if: success()
        run: echo "Deployment successful!"

      - name: Notify on failure
        if: failure()
        run: echo "Deployment failed!"
```

5. Push your code to trigger the deployment:
```bash
git add .
git commit -m "Initial deployment"
git push origin main
```

`Successfully deployed with ci/cd github actions`
![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/a23e2994-0054-41a6-88e9-4ca0208d09b2.png)

`Expose with poirdhi load balancer`

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/2c60e194-57ff-4530-93f6-42699940e74e.png)
