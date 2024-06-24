# Multi-VPC Setup with Node.js and Bastion Hosts
`Basic diagram` 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/6dd7b5eb-a8a9-4079-a793-bb4e93894464.png)
## Overview

This setup involves two VPCs, each with one public and one private subnet. A Node.js server is deployed in the public subnet of the first VPC and in the private subnet of the second VPC. Bastion servers are deployed in the private subnets of both VPCs to allow secure SSH access. The Node.js service in the second VPC calls the service in the first VPC, and the latter traces the NAT IP of the request.




## Steps to Create the Setup

### VPC 1

1. **Create VPC:**
   - CIDR: `10.0.0.0/16`

2. **Create Subnets:**
   - Public Subnet: `10.0.1.0/24`
   - Private Subnet: `10.0.2.0/24`

3. **Create and Attach Internet Gateway**

4. **Create Route Table for Public Subnet and Add Route to Internet Gateway**

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


