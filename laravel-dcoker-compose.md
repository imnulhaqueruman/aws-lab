# Deploy Laravel App on EC2 Instance with Docker Compose

![image](![alt text](image.png))

This comprehensive guide walks you through deploying a Laravel application on AWS EC2 instances using Docker Compose. We'll create a production-ready environment with a custom VPC, properly configured subnets, and security groups. The setup includes separate instances for MySQL and Laravel, containerized with Docker for easy management and scalability.

## Prerequisites
- AWS Account with appropriate permissions
- Basic understanding of Docker and Docker Compose
- Familiarity with Laravel framework
- SSH key pair for EC2 instance access

`After opening the Poridhi lab, click the VS Code icon` 

![mage](https://s3.brilliant.com.bd/blog-bucket/thumbnail/011d3bce-8e13-4fad-bc61-92df67d74c31.png)

`This will open a VS Code server instance for you ` 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/56de24f8-2f4f-4c85-b042-d9fc55f3c2ea.png)

#### Create a Makefile to install Laravel prerequisites ,  clone the Laravel todo app repository and docker in Poridhi VS Code server . 
 
```bash

# Makefile for Laravel Setup

.PHONY: all update install_php install_composer install_nodejs clone_repo setup_permissions setup_env install_assets start_server

# Default task
all: update install_php install_composer install_nodejs clone_repo setup_permissions setup_env install_assets start_server

# Step 1: Update and upgrade the system
update:
	sudo apt update && sudo apt upgrade -y

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

# Step 4: Install Node.js and npm
install_nodejs:
	curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
	sudo apt install -y nodejs

# Step 5: Clone the Laravel project repository
clone_repo:
	git clone https://github.com/imnulhaqueruman/laravel-to-do-app.git project-name

# Step 6: Set proper permissions for Laravel directories
setup_permissions:
	cd laravel-tod-do-app && \
	sudo chown -R $$USER:www-data storage && \
	sudo chown -R $$USER:www-data bootstrap/cache && \
	chmod -R 775 storage && \
	chmod -R 775 bootstrap/cache

# Step 7: Setup environment configuration
setup_env:
	cd laravel-tod-do-app && \
	cp .env.example .env && \
	php artisan key:generate

# Step 8: Install Node.js dependencies and compile assets
install_assets:
	cd laravel-tod-do-app && \
	composer install && \
	npm install && \
	npm run dev

# Step 9: Start the Laravel development server
start_server:
	cd laravel-tod-do-app && \
	php artisan serve
# Install Docker
docker:
	@echo "Installing Docker..."
	sudo apt update -y
	sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
	echo "deb [arch=$(shell dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(shell lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	sudo apt update -y
	sudo apt install -y docker-ce
	sudo systemctl start docker
	sudo systemctl enable docker
	@echo "Docker installed successfully."
```
* Open terminal and run make all 


After runing the php server you will get a error like this 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/d0f419f9-156b-4c85-8307-d3f09129fc98.png) 

`To resolve this issue, you'll need to set up MySQL and update the .env file`

---

## Step 1: VPC and Networking Setup in AWS Console

1. **Create a VPC**:
   - Navigate to **VPC Dashboard** > **Your VPCs** > **Create VPC**.
   - Set the **Name tag** (e.g., `MyAppVPC`).
   - Set **IPv4 CIDR block** to `10.0.0.0/16`.
   - Click **Create VPC**.

2. **Attach an Internet Gateway (IGW)**:
   - Go to **Internet Gateways** > **Create Internet Gateway**.
   - Name the IGW (e.g., `MyAppIGW`) and create it.
   - Select your newly created IGW, click **Actions** > **Attach to VPC**, and choose your VPC (e.g., `MyAppVPC`).

3. **Create Subnets**:
   - Go to **Subnets** > **Create subnet**.
   - Select your VPC (e.g., `MyAppVPC`).
   - Create two subnets, one in each Availability Zone:
     - **Public Subnet** (e.g., `MyAppPublicSubnet`): Use CIDR block `10.0.1.0/24`.
     - **Private Subnet** (optional for future use): Use CIDR block `10.0.2.0/24`.
   - Select each subnet, go to **Actions** > **Modify auto-assign IP settings**, and enable **Auto-assign IP** for the **Public Subnet**.

4. **Create and Configure a Route Table**:
   - Go to **Route Tables** > **Create route table**.
   - Name it (e.g., `MyAppPublicRouteTable`) and select your VPC.
   - Go to **Routes** > **Edit routes** > **Add route**:
     - Set **Destination** to `0.0.0.0/0`.
     - Select **Target** as your IGW (e.g., `MyAppIGW`).
   - Under **Subnet associations**, attach your **Public Subnet** to the route table.

---

## Step 2: Security Group Configuration

1. **Create a Security Group for MySQL**:
   - Go to **Security Groups** > **Create security group**.
   - Name it (e.g., `MySQL-SG`), attach it to your VPC, and add the following rules:
     - **Inbound Rule**: Allow MySQL (port `3306`) access from the Laravel EC2 instance's security group or IP range for restricted access.

2. **Create a Security Group for the Laravel EC2 Instance**:
   - Create another security group (e.g., `Laravel-SG`) for the application instance.
   - Add the following rules:
     - **Inbound Rule**: Allow HTTP (port `8000`) for web traffic from anywhere.
     - **Inbound Rule**: Allow SSH (port `22`) from your IP for management.

---

## Step 3: Launch EC2 Instances for MySQL and Laravel

1. **Launch the MySQL EC2 Instance**:
   - Go to **EC2 Dashboard** > **Instances** > **Launch Instance**.
   - Choose an Amazon Linux 2 or Ubuntu AMI.
   - Select the instance type (e.g., `t2.micro`).
   - Under **Network settings**:
     - Select **MyAppVPC** for the VPC.
     - Choose **MyAppPublicSubnet**.
     - Enable **Auto-assign Public IP**.
   - Under **Configure Security Group**, select **MySQL-SG**.
   - Complete the launch, and note down the public IP address of this instance for database connection configuration.

2. **Launch the Laravel EC2 Instance**:
   - Repeat the steps above, but this time select the **Laravel-SG** security group for the Laravel EC2 instance.

---

### **Step 4: Update and Install MySQL**

`Connect to your MySQL EC2 instance` 

Using SSH, connect to your EC2 instance:

```bash
ssh -i "your-key-pair.pem" ubuntu@<your-ec2-public-dns>
```

Replace `your-key-pair.pem` with your private key file and `<your-ec2-public-dns>` with the instance’s public DNS.

1. **Update packages**:

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Install MySQL Server**:

   ```bash
   sudo apt install mysql-server -y
   ```

3. **Start MySQL Service**:

   ```bash
   sudo systemctl start mysql
   sudo systemctl enable mysql
   ```

### **Step 4: Secure MySQL Installation**

Run the MySQL secure installation script:

```bash
sudo mysql_secure_installation
```

- Follow prompts to set a root password, remove test databases, and disable remote root access.

### **Step 5: Configure MySQL for Remote Access (Optional)**

1. Open the MySQL configuration file:

   ```bash
   sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
   ```

2. Locate the line `bind-address = 127.0.0.1` and change it to:

   ```ini
   bind-address = 0.0.0.0
   ```

3. Save and exit. Restart MySQL to apply changes:

   ```bash
   sudo systemctl restart mysql
   ```

4. Allow a user to connect remotely:

   ```sql
   sudo mysql
   CREATE DATABASE poridhi;
   CREATE USER 'poridhi'@'%' IDENTIFIED BY 'Poridhi@123456';
   GRANT ALL PRIVILEGES ON *.* TO 'poridhi'@'%' WITH GRANT OPTION;
   FLUSH PRIVILEGES;
   ```

## Step 5: Update .env file with database 
 Edit the `.env` file to connect to the MySQL EC2 instance:

   ```env
   DB_CONNECTION=mysql
   DB_HOST=mysql-ec2-instance-ip
   DB_PORT=3306
   DB_DATABASE=laravel_db
   DB_USERNAME=laravel_user
   DB_PASSWORD=laravel_password
   ```
## Step 6 : Build docker image and push to docker hub 

1. `docker login`
2. `docker build -t your-dockerhub-username/laravel-app:latest .`
3. `docker push your-dockerhub-username/laravel-app:latest`

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/e9d76044-9164-4e1f-be8c-6a8591b7bc87.png)

## Step 7: Deploy Laravel on the Laravel EC2 Instance

1. **SSH into the Laravel EC2 instance**:

   ```bash
   ssh -i your-key.pem ec2-user@laravel-ec2-instance-ip
   ```
2. Install docker and docker compose  
   * create a make file to install docker and docker compose  

```bash
   # Makefile for installing Docker and Docker Compose

DOCKER_COMPOSE_VERSION := 2.20.2  # Set your desired Docker Compose version here

.PHONY: all docker docker-compose verify

# Run all steps
all: docker docker-compose verify

# Install Docker
docker:
	@echo "Installing Docker..."
	sudo apt update -y
	sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
	echo "deb [arch=$(shell dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(shell lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	sudo apt update -y
	sudo apt install -y docker-ce
	sudo systemctl start docker
	sudo systemctl enable docker
	@echo "Docker installed successfully."

# Install Docker Compose
docker-compose:
	@echo "Installing Docker Compose version $(DOCKER_COMPOSE_VERSION)..."
	sudo curl -L "https://github.com/docker/compose/releases/download/$(DOCKER_COMPOSE_VERSION)/docker-compose-$(shell uname -s)-$(shell uname -m)" -o /usr/local/bin/docker-compose
	sudo chmod +x /usr/local/bin/docker-compose
	sudo ln -sf /usr/local/bin/docker-compose /usr/bin/docker-compose
	@echo "Docker Compose installed successfully."

# Verify installation
verify:
	@echo "Verifying Docker installation..."
	@docker --version
	@echo "Verifying Docker Compose installation..."
	@docker-compose --version
	@echo "All installations verified successfully."

   ```

3. **Create a Docker Compose file for Laravel**:

   ```yaml
   # /home/ec2-user/sudo nano docker-compose.yml
   version: '3.8'

   services:
     app:
       image: your-dockerhub-username/laravel-app:latest
       ports:
         - "8000:8000"
       environment:
         - APP_ENV=production
         - DB_HOST=mysql-ec2-instance-ip
         - DB_PORT=3306
         - DB_DATABASE=laravel_db
         - DB_USERNAME=laravel_user
         - DB_PASSWORD=laravel_password
   ```

5. **Start Laravel Application**:

   ```bash
   docker-compose up -d
   ```

---

## Step 6: Verify Deployment

1. **Verify MySQL Connection**:
   - From the Laravel container, confirm it can connect to the MySQL instance by running migrations:

     ```bash
     docker-compose exec app php artisan migrate
     ```

2. **Access the Laravel Application**:
   - Go to `http://laravel-ec2-instance-ip` in your browser to see the Laravel app.

---

## Notes

- **Security Group**: Ensure the MySQL EC2 instance only allows inbound connections from the Laravel EC2 instance.
- **Environment Variables**: Replace sensitive data (e.g., passwords) with environment variables for better security.
- **Docker Hub**: Replace `your-dockerhub-username/laravel-app:latest` with your actual Docker image for the Laravel application.

This setup isolates MySQL and Laravel in separate instances within a secure VPC, while Docker Compose simplifies the application management on each EC2 instance.


## Expose your Laravel application to the internet using the Poridhi Load Balancer  

1. Click to load balancer icon in Poridhi lab console, then open a prompt for you 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/7b748f81-2378-457b-86d2-4bfed94bb098.png)

`Paste your Laravel EC2 instance public IP and your container port 8000`

2. After creating load balancer you will get a URL for your Laravel app 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/dd20f3bd-b47a-43ec-89e0-0666dd0f0f3b.png)

Open the URL in your browser and you will see your Laravel app 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/bf93ad61-95f2-4c8a-b0bf-c6620420dba3.png)




