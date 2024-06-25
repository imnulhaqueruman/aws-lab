Sure, I will update the document based on the images provided and check for any grammatical errors. Here's the revised documentation:

---

# Multi-VPC Setup with Node.js and Bastion Hosts

## Overview

This setup involves two VPCs, each with one public and one private subnet. A Node.js server is deployed in the public subnet of the first VPC and in the private subnet of the second VPC. Bastion server-1 is deployed in the private subnets of  VPC-1 and Bastion-server-2 is deployed in Public subnet of VPC-2 . The Node-client-server in the second VPC calls the Node-server in the first VPC, and the Node-server of VPC traces the NAT IP of the request.

## Basic Diagram
```plaintext
       +---------------------------+     +---------------------------+
       |          VPC 1            |     |          VPC 2            |
       |   CIDR: 10.0.0.0/16       |     |   CIDR: 10.1.0.0/16       |
       |                           |     |                           |
       |   +-----------------+     |     |   +-----------------+     |
       |   |  Public Subnet  |     |     |   |  Public Subnet  |     |
       |   |  CIDR: 10.0.1.0/24 |     |     |  CIDR: 10.1.1.0/24 |     |
       |   +-----------------+     |     |   +-----------------+     |
       |         |                 |     |         |                 |
       |   +-----------------+     |     |   +-----------------+     |
       |   | Node-Server  |     |     |   | Bastion Server 2 |     |
       |   +-----------------+     |     |   +-----------------+     |
       |         |                 |     |         |                 |
       |   +-----------------+     |     |   +-----------------+     |
       |   |  Private Subnet |     |     |   |  Private Subnet |     |
       |   |  CIDR: 10.0.2.0/24 |     |     |  CIDR: 10.1.2.0/24 |     |
       |   +-----------------+     |     |   +-----------------+     |
       |         |                 |     |         |                 |
       |   +-----------------+     |     |   +-----------------+     |
       |   | Bastion Server 1 |     |     |   | Node-client-Server  |     |
       |   +-----------------+     |     |   +-----------------+     |
       +---------------------------+     +---------------------------+
```

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/cc7f0cd7-6edb-4f4b-91f9-19dd5f02d244.png)

## Steps to Create the Setup

### VPC 1

1. **Create VPC:**
   - CIDR: `10.0.0.0/16`

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a4c19b14-99ad-488c-8bc2-8b40a515b9c4.png)

2. **Create Subnets:**
   - Public Subnet: `10.0.1.0/24`
   - Private Subnet: `10.0.2.0/24`

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/708ddaad-17a4-4233-b553-952daac70b9b.png)

   **Enable auto-assign public IPv4 in the public subnet**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/dc1b5191-9d03-48f3-92da-9c58ba683ecb.png)

3. **Create and Attach Internet Gateway**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f34eaaf1-6c71-4c11-a2ba-7ba1cf049f8e.png)

4. **Create Route Table for Public Subnet and Add Route to Internet Gateway**

   **Create Public-RT-1**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f9545413-edb5-4289-8a95-bcea25dfe3ce.png)

   **Add Route to IGW-1**

   Select Public-RT-1, scroll down to the Route tab, and set the routes.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/08e828fd-a84d-4f64-a473-62471d2c564e.png)

5. **Associate Public Subnet with Public Route Table**

   Click on the Subnet associations tab, click edit subnet associations, and add the public subnet.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/feffdbe8-4bee-47f4-a80c-d019785e65e1.png)

6. **Create Route Table for Private Subnet**

   Follow the same process to create `Private-RT-1` and edit Subnet Associations.

7. **Create NAT Gateway in Public Subnet and Update Private Route Table**

   On the left-hand menu on the VPC dashboard, click `NAT gateways`, then click the `create nat gateway` button.

   Follow this process to create NAT-1.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/47e1a6a5-58cb-4f8f-ae66-f2b49fb2f3d9.png)

   Go to the Routes table page, select `Private-RT-1`, scroll down to the Routes tab, and click add routes for the NAT gateway.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/b7793693-66d6-4442-85c5-f4a34a791808.png)

8. **Create Security Group SG-Node-1 for Node Server in Public Subnet**

   Select Security Group from the VPC dashboard and click `create security group`.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a0ff24bf-4a57-4110-842e-e2619c4ea436.png)

9. **Create Security Group for Bastion Server-1 in Private Subnet**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/b9906b6f-452b-4a33-9ae5-d79d7db9df82.png)

10. **Deploy Instances:**
    - **Node.js Server in Public Subnet**

      Launch Node-server-1 with these network settings.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/1d5a8ea9-6774-4f43-87a7-ba3d109db850.png)

      **Connect Node-server-1**

```bash
cd Downloads/
chmod 400 "Node-1.pem"
ssh -i "Node-1.pem" ubuntu@<Public_IP_of_Node-1>
```

* Install Node.js with the following commands:

```bash
touch .bashrc
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
. /.nvm/nvm.sh
nvm install --lts 

# Forward port 80 traffic to port 3000
echo "Forwarding 80 -> 3000"
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 3000

# Install & use pm2 to run Node app in background
echo "Installing & starting pm2"
sudo apt install npm
npm install pm2@latest -g
pm2 start index.js
```

* Initialize the project

```bash 
mkdir vpc1-service
cd vpc1-service
npm init -y
npm install express
```

* Create Server script

```bash 
sudo nano index.js 
```

* Paste this code

```js
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  const clientIp = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
  console.log(`Request received from IP: ${clientIp}`);
  res.send('Hello from VPC1!');
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

You can use pm2 to run the Node server in the background. To monitor the log of your service you can use this command:

```bash
pm2 monitor
```

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a5d5bfa6-383b-4b9c-b0bb-22498e1df70a.png)

* **Bastion Server in Private Subnet**

   Launch Bastion Server using the same key pair with the following settings.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/c9cf129e-5ee2-4750-9744-b3c5e04f0592.png)

Since the Bastion server is launched in a private subnet, you can't SSH into it from outside. First, SSH into the Node server and paste the key pair of node-server-1 from your local folder.

```bash 
sudo nano Node-1.pem 
# paste your bastion key
```

* SSH into the Bastion server from Node-1 server.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f54debcc-0a00-47cd-a403-ef4582cdd155.png)

### VPC 2

1. **Create VPC:**
   - CIDR: `10.1.0.0/16`

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f8312f10-d775-4fc5-9238-7e9e7a2b1173.png)

2. **Create Subnets:**
   - Public Subnet: `10.1.1.0/24`
   - Private Subnet: `10.1.2.0/24`

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/ba4d94c0-d071-476c-810c-0fa3c4919302.png)

   **Enable auto-assign public IPv4 in the public subnet**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/1e321e3c-0887-41f1-b4af-97b4adf52a28.png)

3. **Create and Attach Internet Gateway**

   Create `IGW-2` and attach it to VPC-2.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/4444610e-4cad-4c03-9595-6979ff961a0e.png)

4. **Create Route Table for Public Subnet and Add Route to Internet Gateway**

   Follow the previous process to create a Public-RT-2 for VPC-2 and add a route for internet gateway IGW-2.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/7fe038fc-e2f1-493e-8f52-81c03a25b135.png)

5. **Associate Public Subnet with Public Route Table**

   Associate the Public subnet with the Public route.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/33473749-c499-4fe3-beb5-2c4e2009ad37.png)

6. **Create Route Table for Private Subnet**

   Follow the previous process to create Private-RT-2 and associate it with Private-sub-2.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/53f0e96c-33d4-4085-aa46-982a1b504400.png)

7. **Create NAT Gateway in Public Subnet and Update Private Route Table**

   Follow the previous process to create another NAT gateway for VPC-2.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/1f6a7093-8b6d-4871-90ea-abd4deed377b.png)

   Edit Routes for Private-RT-2 to allow the destination of VPC-1 and the internet.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a4937181-ead6-4ca0-8c97-25ae70e9af1b.png)

8. **Create Security Group for Bastion-2**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/fc277d4a-0494-46c8-b6dc-386b0dd468d0.png)

9. **Create Security Group for Node-2**

   Allow VPC-2 IP for SSH into it.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/d900b69c-1d5e-452f-899a-68bdf6d7ebdb.png)

10. **Deploy Instances:**
    - **Bastion Server in Public Subnet**

      Launch bastion-server-2 in Public subnet with the following network settings.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/8f583998-d7fd-4782-a1fb-f597e840a411.png)

 **Connect bastion-2 server**

```bash 
cd Downloads 
chmod 400 "bastion-2.pem"
ssh -i "bastion-2.pem" ubuntu@<Public_IP_of_Bastion-2>
```

    - **Launch Node-server-2 in Private Subnet**

      Launch Node-server-2 with the same key pair and the following network settings.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/2a221dd1-3b06-461a-88c8-1bb8bce228ed.png)

      Since Node-server-2 is launched in a private subnet, you can't SSH into it from outside. First, SSH into bastion-2 server and paste the key pair of node-server-1 from your local folder.

```bash
sudo nano bastion-2.pem 
#paste your key pair 
```

* Connect Node-server-2 from bastion-2 server:

```bash 
ssh -i "bastion-2.pem" ubuntu@10.1.2.206
```

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f17a320e-ea69-4bae-870e-8ab2fc60ba4d.png)

* Install Node.js and npm following the previous process.

* Initialize Node project for Node-server-2:

```bash 
mkdir vpc2-service
cd vpc2-service
npm init -y
npm install axios
```

* Create the client script (client.js):

```js
const axios = require('axios');

const callService = async () => {
  try {
    const response = await axios.get('http://<VPC1_Node_Server_PUBLIC_IP>:3000');
    console.log('Response from VPC1:', response.data);
  } catch (error) {
    console.error('Error calling VPC1 service:', error.message);
  }
};

callService();
```

**Enable PM2 monitor in Node-server-1 of VPC-1**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a5d5bfa6-383b-4b9c-b0bb-22498e1df70a.png)

**Request to Node-server-1 of VPC-1 from Node-client-server of VPC-2**

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/0e346310-b67a-49d4-959c-7d4cdb69dafb.png)

* Open PM2 monitoring web URL and see the results.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/9eb2c390-813c-4d16-a5b2-c215ff7302b3.png)

Here you can see Node-server-1 traces the request IP of Node-client-server of VPC2. This trace IP is NAT-2's public IP of the Public subnet of VPC-2.

