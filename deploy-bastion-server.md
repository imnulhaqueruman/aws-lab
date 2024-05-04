# Deploy a Bastion server in Public Subnet . 

###### When it comes to keeping cloud data safe, AWS offers a Virtual Private Cloud (VPC) service. This lets users set up a private network within AWS. A key security feature of VPC is the bastion server. A bastion host is a server that is deployed within the VPC environment, acting as a gateway between the internet and the private VPC environment. In this lab we deploy a bastion server in public subnet. 


# What is Bastion Server? 

A bastion server is an EC2 instances that act as a secure gateway for AWS VPC environment to access private instances . It's also called a jump server because it is used to jump from the internet into private network .

# How does Bastion server work? 

A bastion server runs on an Ubuntu EC2 instances hosted in a Public subnet of AWS VPC . This instance set up with a security group that allows SSH to it from anywhere over the internet .

# Architecture 

This diagram illustrates setting up a bastion host in AWS VPC cloud. The bastion host operates on an Ubuntu EC2 instance and is usually placed in the public subnet of VPC.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/2cde82fd-4801-4c68-8269-d93affe7e805.png)

## Let’s head over to the AWS Console.

### Step 1: Create a Custom VPC with a CIDR of 10.0.0.0/16

Navigate VPC page from AWS console.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/613aec45-bc73-4e46-9503-845178331bb1.png)

On VPC page, In left hand menu Select your VPCs, Then click on Create VPC . 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/ec52a4f2-f13e-456d-8e6b-940ce465a269.png)

```bash 
Note: Do not use the VPC Wizard to create your VPC; instead, configure your VPC from scratch and use the VPC Only option.

```
* Create VPC 

1. VPC Name :` Poridhi `
2. IPv4 CIDR block: `10.0.0.0/16`
3. IPv6 CIDR block: No IPv6 CIDR block
4. Tenancy: Default Tenancy

Then click Create VPC at the bottom of the page .


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a9e9a7f9-a689-44b7-bb2f-c7ce55f8b4ac.png)

Hurrah!! Successfully created A VPC name Poridhi in AWS. 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/756416ff-d491-4064-bcb4-1b099be1eda8.png)

### Step 2: Create a Public Subnet with a CIDR of 10.0.1.0/24

On VPC dashboard find `Subnets` in the left hand side menubar , once there click create subnet. 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/0426b62c-cdfc-4e70-bb74-376b6f4e9b59.png)

First things select the  `VPC ID `, which , if you click into the drop down menu .

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/b5e42db5-5686-430e-917d-d192a6ae9d71.png)

 ####  create a Public subnet.
 *  Under the Subnet settings,

    1. Subnet Name: Type poridhi-subnet 

    2. Availability Zone: Select ap-southeast-1a

    3. IPv4 CIDR block: Type 10.0.1.0/24

Then click create subnet at the bottom of the page

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/e798455c-538b-4c16-a0ca-ca114a0cc681.png)

### Assign Public IPv4 address

 1. From the Subnets page, in the left-hand menu, select Subnets.


 2. On the Subnets page, click the checkbox next to  subnet to   select it.


 3. In the top right-hand corner of the page, click Actions > Edit subnet settings.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/ead97061-eb10-4aee-8889-77606f4b34d8.png)


 4. On the Edit subnet settings page, under Auto-assign IP settings, clickthe  checkbox next to Enable auto-assign public IPv4 address.

  5. Click Save at the bottom of the page.


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/730f2c5b-ce3e-4531-8046-fa8e52ce2471.png)




### Step 3: Create and Attach the Internet Gateway with VPC 

On the left side of VPC dashboard locate Internet Gateway on VPC side bar menu . Then click create internet gateway 


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/c41132ba-e961-42dd-914a-cc3adfaf84c7.png)


On the Create internet gateway page, under Name tag, type `poridhi-igw`. Then click Create internet gateway at the bottom of the page.


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/2cda9098-9bb1-487f-a754-94ba17607a85.png)


After successfully created internet gatway , then click on Actions menubar to Attach `poridhi-igw` with `Poridhi` VPC 


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/95041602-805f-476d-b05d-c9589ca478d9.png)


In the drop-down menu, select the VPC that you've created. Then, choose "Attach internet gateway" to connect the internet gateway to the selected VPC.


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/245d2759-3fae-46f0-9367-ee465b6c9b40.png)



### Create and Connect the Route Tables

From the internet gateway’s page, in the left-hand menu on VPC dashboard, select Route tables and you'll see a default route table has already been built.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/ba86e1d5-dd34-438b-8006-4b3893e7edb7.png) 


  * Click Create route table in the top right.

  * On the Create route table page, under Name — optional, type the name `poridhi-public-route`
   
  * Under VPC, click into the empty field, and then select the one (poridhi) option.

  * Click Create route table at the bottom of the page.


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/3c585a6b-7cda-4e7e-bc18-8622992c61cd.png)

#####  Now need to edit route for the poridhi-public-route to connect with IGW . 


On the route table’s page, halfway down the page, on the Routes tab, click  Edit routes on the right.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/c881ef36-12f5-465b-b36b-0db9aa754a13.png)


   * On the Edit routes page, click Add Route.

   * For the new route, under Destination, type 0.0.0.0/0.

   * Under Target, select Internet Gateway, and then click again on the igw- that ends with (poridhi-igw).

   * Click Save changes.

 ![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/754fb9f1-0aba-45de-b7a1-91f74ae87eb5.png)


### Step 4: Subnet Associations 

 * On the route table’s page, halfway down the page, click the Subnet associations tab.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/3d73e8a0-44fd-479b-af78-d7800ac7405a.png)


 * On the Subnet associations tab, in the Explicit subnet associations section, click Edit subnet associations.

 * Click the checkbox next to the poridhi-subent-public subnet.

 * Click save associations 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/bb6adc63-906b-43da-a4e5-593c067f0931.png)


 ### Step 5: Create Security Groups for public Subnet 

 On the VPC dashboard page locate secuirty groups , then click create secuirty groups at the top right corner of Security groups page. 

 ![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/853306b3-7644-4127-bdc8-bacf68504bae.png)

 * Secuirty Group name : porihdi-sg 

 * Description: Allow for SSH 

 * VPC : Select VPC(poridhi) from drop down list 

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/951d3f70-e35c-4508-b350-128faccec75c.png)

* `Inbound Rules `: Allow inbound SSH connection from anywhere i.e. 0.0.0.0/0 (internet).


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/895aa622-179c-4b17-8cb7-cbc5c012de8c.png)


* `Outbound Rules`: Allow outbound SSH connection to network IP addresses within the VPC CIDR range i.e. 10.0.0.0/16 (default VPC).

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/ecfea234-740a-4743-b84f-5629f5dd5648.png)


### Step 6: Launch EC2 instance in public subnet 

 Let's create some EC2 instances. 

 Let's navigate ec2 instances console and clicl instances.Then click launch instances .

1. From the route table’s page, navigate to the EC2 service by typing EC2 into the search bar at the top of the page and selecting EC2 from the dropdown.

2. In the left-hand menu, click Instances.

3. On the Instances page, select Launch instances in the top right.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/bb236cfe-1398-4baa-9fc6-651d5f0e001e.png)

4. On the Launch an instance page, in the name field, type `poridhi-one-ec2`.

5. In the Application and OS Images section, make sure, by default, that Ubuntu Server 24.04 LTS and make sure the Architecture is set to 64-bit (x86).

6. In the Instance Type section, select t3.micro from the dropdown.

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/51b347b6-e18d-41d6-861b-9b6745648831.png)

7. In the Key pair (login) section, click Create new key pair.

8. In the Create key pair window, for the Key pair name, type `poridhi-one`.

9. Leave the Key pair type set to RSA.

10. For the private key file format, choose either .pem or .ppk, depending on if you will use PuTTY or OpenSSH to connect to the EC2 instance later on.
Note: Use .pem for a mac device; use .ppk for a Windows device.

11. Click Create key pair.
 
![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/f9ed25a0-9456-4e73-b34c-ff981509e7a9.png)

12. Back on the Launch an instance page, in the Network settings section, click Edit.

13. Make sure that, by default, the VPC is set to a vpc- that ends in
 (poridhi), and the Subnet is set to the `poridhi-subnet`  subnet.

14. Select exesiting security groups (poridhi-sg)

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/96726a4e-c0cc-49b8-acd3-05dacc3ff6ff.png)

* Leave all other settings as the default and click Launch instance.
Note: Your `poridhi-one.pem` or .ppk key pair file should have downloaded. You will use this file later in the lab.

* Go back to the Instances section of the EC2 dashboard.

* On the Instances page, hit the refresh icon at the top of the screen to check the progress of the poridhi-one-ec2.



![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/58cad544-3882-4f45-83df-b0db2cdb7ca0.png)


### Step 7 : Connect Bastion server from User end . 

   1. From the list of instances in the EC2 console, right-click on the `poridhi-one-ec2` and click Connect.

   2. On the Connect to instance page, select the SSH client tab.

   3. On the SSH client tab, directly underneath Example:, copy the connection command to your clipboard. (You can optionally paste it into a text editor to keep it handy.)

   4. Open your SSH client.

   5. Locate your private key file on your local machine that you downloaded earlier, `poridhi-one.pem `(or .ppk). If it is in your local Downloads folder, you  can run cd ~/Downloads.

   6. Run the command chmod 400  `poridhi-one.pem `, to ensure your key is not publicly viewable.


![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/61e3400d-f89a-4dc8-af35-a2262c5ccab8.png)



   1. Connect to the public instance by running the connection command you copied to your clipboard earlier.

   2. Type in yes to the connection prompt. You should be able to connect.

Here ubuntu@44.208.34.227 is the Bastion server .

![image](https://blog-bucket.s3.brilliant.com.bd/thumbnail/1e4fb7d2-208e-49a1-9d06-332b56e138e1.png) 






