# Multi-VPC Setup with Node.js and Bastion Hosts
`Basic diagram` 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/6dd7b5eb-a8a9-4079-a793-bb4e93894464.png)
## Overview

This setup involves two VPCs, each with one public and one private subnet. A Node.js server is deployed in the public subnet of the first VPC and in the private subnet of the second VPC. Bastion servers are deployed in the private subnets of both VPCs to allow secure SSH access. The Node.js service in the second VPC calls the service in the first VPC, and the latter traces the NAT IP of the request.




## Steps to Create the Setup

### VPC 1

1. **Create VPC:**
   - CIDR: `10.0.0.0/16`
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a4c19b14-99ad-488c-8bc2-8b40a515b9c4.png)

2. **Create Subnets:**
   - Public Subnet: `10.0.1.0/24`
   - Private Subnet: `10.0.2.0/24`
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/708ddaad-17a4-4233-b553-952daac70b9b.png)

  **Enable public Ipv4 in public subnet** 
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/dc1b5191-9d03-48f3-92da-9c58ba683ecb.png)

3. **Create and Attach Internet Gateway**
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f34eaaf1-6c71-4c11-a2ba-7ba1cf049f8e.png)

4. **Create Route Table for Public Subnet and Add Route to Internet Gateway**

**Create Publict-RT-1**
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f9545413-edb5-4289-8a95-bcea25dfe3ce.png)

**Add Route to IGW-1** 
Select Public-RT-1 and scroll down half of the page then click on Route tabs and set the routes 
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/08e828fd-a84d-4f64-a473-62471d2c564e.png)

5. **Associate Public Subnet with Public Route Table**

6. **Create Route Table for Private Subnet**

7. **Create NAT Gateway in Public Subnet and Update Private Route Table**

8. **Deploy Instances:**
   - Node.js Server in Public Subnet
   - Bastion Server in Private Subnet

### VPC 2

1. **Create VPC:**
   - CIDR: `10.1.0.0/16`

2. **Create Subnets:**
   - Public Subnet: `10.1.1.0/24`
   - Private Subnet: `10.1.2.0/24`

3. **Create and Attach Internet Gateway**

4. **Create Route Table for Public Subnet and Add Route to Internet Gateway**

5. **Associate Public Subnet with Public Route Table**

6. **Create Route Table for Private Subnet**

7. **Create NAT Gateway in Public Subnet and Update Private Route Table**

8. **Deploy Instances:**
   - Node.js Server in Private Subnet
   - Bastion Server in Private Subnet

### VPC Peering

1. **Create VPC Peering Connection:**
   - Between VPC1 and VPC2

2. **Update Route Tables for Peering:**
   - VPC1 to VPC2 CIDR
   - VPC2 to VPC1 CIDR

### Security Groups Configuration

1. **Update Security Groups to Allow Necessary Traffic:**
   - HTTP from VPC2 Private Subnet to VPC1 Public Subnet Node.js Server
   - SSH from Bastion in VPC2 to Node.js Server in VPC1

### Verify and Deploy Node.js Application

1. **Deploy Node.js Application on All Servers**
2. **Ensure VPC2 Node.js Server Calls VPC1 Node.js Server**
3. **Configure VPC1 Node.js Server to Log Source IP**

## Conclusion

By following these steps, you will create a multi-VPC environment with secure communication between services and bastion hosts for secure SSH access. The Node.js service in the second VPC will be able to call the service in the first VPC, and the latter will trace the NAT IP of the request.


