


# Setting Up PostgreSQL on an EC2 Instance (Ubuntu)

This guide will walk you through the steps to install and configure PostgreSQL on an EC2 instance running Ubuntu.

## Step 1: Launch an EC2 Instance
1. Log in to your [AWS Management Console](https://aws.amazon.com/console/).
2. Navigate to the **EC2 Dashboard** and click on **Launch Instance**.
3. Choose an **Ubuntu Server AMI**.
4. Follow the instance creation wizard:
   - Select instance type.
   - Configure instance details.
   - Add storage as needed.
   - Configure security groups:
     - Ensure inbound rules allow connections on port **5432** (PostgreSQL default port).
5. Launch the instance.

## Step 2: Connect to Your EC2 Instance
Once your instance is running, connect via SSH from your terminal:
```bash
ssh -i your-key.pem ubuntu@<your-ec2-instance-ip>
```

## Step 3: Update Your System
Ensure the package list is up to date and upgrade the installed packages:
```bash
sudo apt update
sudo apt upgrade
```

## Step 4: Install PostgreSQL
Install PostgreSQL and the necessary extensions:
```bash
sudo apt install postgresql postgresql-contrib
```

## Step 5: Configure PostgreSQL
### Switch to the PostgreSQL User
```bash
sudo -u postgres psql
```

### Create a PostgreSQL User and Database
1. Create a new PostgreSQL user with a password:
   ```sql
   CREATE USER yourusername WITH PASSWORD 'yourpassword';
   ```

2. Create a new database and grant privileges to the user:
   ```sql
   CREATE DATABASE yourdatabase;
   GRANT ALL PRIVILEGES ON DATABASE yourdatabase TO yourusername;
   ```

3. Exit the PostgreSQL prompt:
   ```bash
   \q
   ```

## Step 6: Configure PostgreSQL for Remote Access (Optional)
By default, PostgreSQL only allows local connections. To allow remote access:

1. Edit the PostgreSQL configuration file:
   ```bash
   sudo nano /etc/postgresql/<version>/main/postgresql.conf
   ```
2. Find the `listen_addresses` setting and change it to allow connections from any IP:
   ```bash
   listen_addresses = '*'
   ```

3. Edit the `pg_hba.conf` file to define allowed connections:
   ```bash
   sudo nano /etc/postgresql/<version>/main/pg_hba.conf
   ```
4. Add the following line to allow connections from all IPs (for testing purposes; use caution in production):
   ```bash
   host    all             all             0.0.0.0/0            md5
   ```

## Step 7: Restart PostgreSQL
To apply the changes, restart PostgreSQL:
```bash
sudo service postgresql restart
```

## Step 8: Configure Firewall/Security Group
Ensure that your EC2 instance's security group allows inbound traffic on port **5432** (PostgreSQL's default port):
1. Go to the **EC2 Dashboard** > **Security Groups**.
2. Edit the inbound rules of the security group attached to your instance.
3. Add a rule for PostgreSQL:
   - **Type**: PostgreSQL
   - **Port**: 5432
   - **Source**: Custom (restrict to trusted IP addresses for security)

## Step 9: Access PostgreSQL
You can now connect to your PostgreSQL database using:
- **psql** command-line tool:
  ```bash
  psql -h <your-ec2-instance-ip> -U yourusername -d yourdatabase
  ```
- A graphical client like **pgAdmin**.

---

Now PostgreSQL is ready on your EC2 instance! Make sure to properly secure your database before using it in production.

---

### **Step 1: Create an ECS Cluster**
1. **Navigate to the ECS Dashboard** in the AWS Management Console.
2. Click on **Get Started** and select **Create Cluster**.
3. **Name** your cluster (e.g., `example-cluster`).

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/657a25cd-a06b-44a9-b35e-722b4e49bdd3.png)

4. Select **Amazon EC2 instance** as the launch type.
5. Choose **Amazon Linux 2** for the OS.
6. Choose **t2.micro** as the instance type (for free tier).
7. Leave the rest of the settings as default.
8. Click **Create** to complete the cluster creation.


![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/766451bd-f9b1-4b67-a91c-ba201386195e.png)

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/1b135ac2-4ccd-47a6-a429-45042f4ed0e1.png)

---

### **Step 2: Create a Target Group**
1. Open the **EC2 Dashboard** in the AWS Management Console.
2. In the left-hand menu, under **Load Balancing**, click **Target Groups**.
3. Click **Create Target Group**.
4. Select **Instance** as the target type.
5. Provide a name for your target group (e.g., `example-tg`).
6. Set the **protocol** to **HTTP** and the **port** to match your Docker container’s port (e.g., 8080).
7. In the **Health check** section, set the path to `/health`.

Here’s a simple health check endpoint using Node.js and Express:

```javascript
const express = require('express');
const app = express();

app.use('/health', (req, res) => {
  res.status(200).send('OK');
});

const port = 8080;
app.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

---

### **Step 3: Create an Application Load Balancer (ALB)**
1. Go to **Load Balancers** under the **EC2** section.
2. Click **Create Load Balancer** and select **Application Load Balancer**.
3. Provide a unique name (e.g., `example-lb`).
4. Under **Network Mapping**, select at least two subnets for redundancy.
5. In the **Listeners** section, select **HTTP** and port **8080**.
6. In the **Routing** section, select the target group you created earlier (`example-tg`).
7. Click **Create**.

---

### **Step 4: Create an ECS Task Definition**
1. Navigate to the **ECS Dashboard** and go to **Task Definitions**.
2. Click **Create New Task Definition**.
3. Choose **EC2** as the launch type.
4. Provide a name for the task (e.g., `example-t`).
5. Select **Bridge** as the network mode if your container needs to communicate externally.
6. In the **Container Definitions** section:
   - Provide a name for the container (e.g., `example-container`).
   - Enter the **Image URI** of your Docker image (e.g., `docker.io/your-username/example-app:latest`).
   - Set the **port mappings** (host port 8080 to container port 8080).
7. Add any necessary environment variables for your application.
8. Click **Create** to finish creating the task definition.

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/14e1b226-531e-4446-a74b-79b41c355f94.png)

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/d7459b37-9069-4953-90f4-c42a6d0a5600.png)
---

### **Step 5: Create an ECS Service**
1. Go to your **ECS cluster** and click on the **Services** tab.
2. Click **Create**.
3. In the **Task Definition** section, select your task definition and revision (e.g., `example-t`).
4. Provide a name for the service (e.g., `example-s`).
5. Set the desired number of tasks (e.g., `1`).
6. Under the **Load Balancing** section, choose your previously created Application Load Balancer (`example-lb`).
7. Select the existing target group (`example-tg`).
8. Set the **Health check grace period** to 120 seconds.
9. Click **Create Service** to complete the setup.

---

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/d7cfe561-9787-4a7f-b3b5-1824323f8df1.png)
![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/264c3ac5-6eb4-4b05-a62f-28325450f771.png)


`Check task overview after runing task `

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/33083062-d779-4aa8-bf91-852627767a5a.png)



### **Step 6: Create a GitHub Actions Workflow**

To automate your deployment process using GitHub Actions, follow these steps:

1. **Navigate to your GitHub repository.**
2. Create a directory named `.github/workflows`.
3. Inside that directory, create a file named `deploy.yml`.

4. In `deploy.yml`, add the following content:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '14'

      - name: Install dependencies
        run: npm install

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/nodeecs:v1.6

      - name: Deploy to ECS
        run: |
          aws ecs update-service --cluster node-cluster --service update-svc --force-new-deployment
```

**Note**: You’ll need to set the following values as GitHub secrets in your repository settings:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `DOCKER_TOKEN`
- `DOKCER_USERNAME`

---

# Expose with Poridhi Loab balancer 

1. Copy Public IP from your ECS instance 

2. In Enter IP section Paster your public instance public IP and http listen port 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/1967fc53-92bb-4b74-9603-4160b4446c9e.png)

3. Copy this url 

![image](https://s3.brilliant.com.bd/blog-bucket/thumbnail/b3ed68fe-46f3-4a32-b13e-6e8ccb16a3da.png)



