# Pulumi AWS Infrastructure Setup Guide

## Prerequisites
- Python 3.7 or later
- Pulumi CLI installed
- AWS CLI installed and configured
- AWS account with appropriate permissions
- SSH key pair generated (`~/.ssh/ecs-key.pub`)

## Infrastructure Components
This Pulumi project sets up:
1. VPC with public and private subnets
2. ECS cluster with EC2 instances
3. Application Load Balancer
4. MySQL database
5. Security groups and networking components
6. Auto Scaling Group for ECS

## Setup Instructions

### 1. Initial Setup
```bash
# Install Pulumi CLI (if not already installed)
curl -fsSL https://get.pulumi.com | sh

# Install required Python packages
pip install pulumi pulumi-aws

# Create new Pulumi project
mkdir pulumi-aws-infra && cd pulumi-aws-infra
pulumi new aws-python
```

### 2. SSH Key Setup
```bash
# Generate SSH key pair if not exists
ssh-keygen -t rsa -b 2048 -f ~/.ssh/ecs-key
```

### 3. Environment Configuration
Create a `Pulumi.dev.yaml` file with the following configuration:

```yaml
config:
  aws:region: ap-northeast-3
  project:environment: dev
```

### 4. Required IAM Roles
Ensure you have created the following IAM role:
- `ecsTaskExecutionRole` with policies:
  - `AmazonECSTaskExecutionRolePolicy`
  - `AmazonEC2ContainerServiceforEC2Role`

### 5. Database Configuration
Update the MySQL instance configuration in the code:
- Update the AMI ID if needed
- Modify the user data script for MySQL installation
- Update the security group rules if needed

### 6. Application Configuration
Update the ECS task definition environment variables:
```python
"environment": [
    {"name": "DB_HOST", "value": "<your-mysql-instance-ip>"},
    {"name": "DB_USERNAME", "value": "<your-db-username>"},
    {"name": "DB_PASSWORD", "value": "<your-db-password>"},
    # ... other environment variables
]
```

### Pulumi code 

```python
import pulumi
import pulumi_aws as aws
import json
import os
import base64
from pulumi_aws import ec2


with open(os.path.expanduser('~/.ssh/ecs-key.pub')) as f:
    public_key = f.read()

# Create the key pair
ecs_key_pair = aws.ec2.KeyPair("ecs-key-pair",
    public_key=public_key,
    tags={
        "Name": "ecs-key-pair"
    }
)

# VPC Configuration
vpc = aws.ec2.Vpc("main-vpc",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    enable_dns_support=True,
    tags={
        "Name": "main-vpc"
    }
)

# Internet Gateway
igw = aws.ec2.InternetGateway("main-igw",
    vpc_id=vpc.id,
    tags={
        "Name": "main-igw"
    }
)

# Public Subnets
public_subnet_1 = aws.ec2.Subnet("public-subnet-1",
    vpc_id=vpc.id,
    cidr_block="10.0.1.0/24",
    availability_zone="ap-northeast-3a",
    map_public_ip_on_launch=True,
    tags={"Name": "public-subnet-1"}
)

public_subnet_2 = aws.ec2.Subnet("public-subnet-2",
    vpc_id=vpc.id,
    cidr_block="10.0.2.0/24",
    availability_zone="ap-northeast-3b",
    map_public_ip_on_launch=True,
    tags={"Name": "public-subnet-2"}
)

public_subnet_3 = aws.ec2.Subnet("public-subnet-3",
    vpc_id=vpc.id,
    cidr_block="10.0.3.0/24",
    availability_zone="ap-northeast-3c",
    map_public_ip_on_launch=True,
    tags={"Name": "public-subnet-3"}
)

# Private Subnets
private_subnet_1 = aws.ec2.Subnet("private-subnet-1",
    vpc_id=vpc.id,
    cidr_block="10.0.4.0/24",
    availability_zone="ap-northeast-3a",
    tags={"Name": "private-subnet-1"}
)

private_subnet_2 = aws.ec2.Subnet("private-subnet-2",
    vpc_id=vpc.id,
    cidr_block="10.0.5.0/24",
    availability_zone="ap-northeast-3b",
    tags={"Name": "private-subnet-2"}
)

# NAT Gateway EIP
# NAT Gateway EIP
nat_eip = aws.ec2.Eip("nat-eip",
    domain="vpc",  # Change vpc=True to domain="vpc"
    tags={"Name": "nat-eip"}
)

# NAT Gateway
nat_gateway = aws.ec2.NatGateway("main-nat-gateway",
    allocation_id=nat_eip.id,
    subnet_id=public_subnet_1.id,
    tags={"Name": "main-nat-gateway"}
)

# Route Tables
public_route_table = aws.ec2.RouteTable("public-rt",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            gateway_id=igw.id,
        )
    ],
    tags={"Name": "public-rt"}
)

private_route_table = aws.ec2.RouteTable("private-rt",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            nat_gateway_id=nat_gateway.id,
        )
    ],
    tags={"Name": "private-rt"}
)

# Route Table Associations - Public
public_rt_assoc_1 = aws.ec2.RouteTableAssociation("public-rt-assoc-1",
    subnet_id=public_subnet_1.id,
    route_table_id=public_route_table.id
)

public_rt_assoc_2 = aws.ec2.RouteTableAssociation("public-rt-assoc-2",
    subnet_id=public_subnet_2.id,
    route_table_id=public_route_table.id
)

public_rt_assoc_3 = aws.ec2.RouteTableAssociation("public-rt-assoc-3",
    subnet_id=public_subnet_3.id,
    route_table_id=public_route_table.id
)

# Route Table Associations - Private
private_rt_assoc_1 = aws.ec2.RouteTableAssociation("private-rt-assoc-1",
    subnet_id=private_subnet_1.id,
    route_table_id=private_route_table.id
)

private_rt_assoc_2 = aws.ec2.RouteTableAssociation("private-rt-assoc-2",
    subnet_id=private_subnet_2.id,
    route_table_id=private_route_table.id
)

# Security Groups
app_security_group = aws.ec2.SecurityGroup("laravel-sg",
    description="Security group for application servers",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=22,
            to_port=22,
            cidr_blocks=["0.0.0.0/0"],
            description="SSH"
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=80,
            to_port=80,
            cidr_blocks=["0.0.0.0/0"],
            description="HTTP"
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=8000,
            to_port=8000,
            cidr_blocks=["0.0.0.0/0"],
            description="Custom TCP 8000"
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=3306,
            to_port=3306,
            cidr_blocks=["0.0.0.0/0"],
            description="MySQL"
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=2049,
            to_port=2049,
            cidr_blocks=["10.0.0.0/16"],
            description="ECS Agent"
        ),
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            protocol="-1",
            from_port=0,
            to_port=0,
            cidr_blocks=["0.0.0.0/0"],
        )
    ],
    tags={"Name": "laravel-sg"}
)

alb_security_group = aws.ec2.SecurityGroup("alb-sg",
    description="Security group for ALB",
    vpc_id=vpc.id,
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=80,
            to_port=80,
            cidr_blocks=["0.0.0.0/0"],
            description="HTTP"
        ),
         aws.ec2.SecurityGroupIngressArgs(
            protocol="tcp",
            from_port=443,
            to_port=443,
            cidr_blocks=["0.0.0.0/0"],
            description="HTTPS"
        ),
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            protocol="-1",
            from_port=0,
            to_port=0,
            cidr_blocks=["0.0.0.0/0"],
        )
    ],
    tags={"Name": "alb-sg"}
)

# Application Load Balancer
alb = aws.lb.LoadBalancer("app-alb",
    internal=False,
    load_balancer_type="application",
    security_groups=[alb_security_group.id],
    subnets=[
        public_subnet_1.id,
        public_subnet_2.id,
        public_subnet_3.id
    ],
    tags={"Name": "app-alb"}
)

# Target Group
target_group = aws.lb.TargetGroup(
    "app-tg",
    port=80,  # Example port, replace with your actual configuration
    protocol="HTTP",  # Example protocol, replace if different
    target_type="ip",  # Change to "ip" for awsvpc mode compatibility
    vpc_id=vpc.id  # Replace with the actual VPC ID variable if needed
)

# ALB Listener
listener = aws.lb.Listener("app-listener",
    load_balancer_arn=alb.arn,
    port=80,
    default_actions=[{
        "type": "forward",
        "target_group_arn": target_group.arn
    }]
)


# EC2 Instances
ubuntu_instance = aws.ec2.Instance("ubuntu-server",
    instance_type="t2.micro",
    ami="ami-05f4d8898209c4f55",
    subnet_id=public_subnet_1.id,
    vpc_security_group_ids=[app_security_group.id],
    tags={"Name": "ubuntu-server"},
    key_name=ecs_key_pair.key_name,
    opts=pulumi.ResourceOptions(depends_on=[ecs_key_pair])  # Add this line
)

amazon_linux_instance = aws.ec2.Instance("amazon-linux-server",
    instance_type="t3.medium",
    ami="ami-04ef3193cda2ac9d4",
    subnet_id=public_subnet_2.id,
    vpc_security_group_ids=[app_security_group.id],
    tags={"Name": "amazon-linux-server"},
    key_name=ecs_key_pair.key_name,
    opts=pulumi.ResourceOptions(depends_on=[ecs_key_pair])  # Add this line
)

# MySql deployment 

# Configuration values
vpc_id = vpc.id  # Existing VPC ID where you want to deploy MySQL
subnet_id = public_subnet_1.id  # Subnet ID for the EC2 instance
instance_type = "t2.micro"  # Instance type for the EC2 instance
mysql_port = 3306  # MySQL default port



# Create an EC2 Instance with MySQL installation
user_data_script = """#!/bin/bash
sudo apt update && sudo apt upgrade -y
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo systemctl enable mysql
"""

ec2_instance = ec2.Instance(
    "mysql-instance",
    instance_type=instance_type,
    subnet_id=subnet_id,
    vpc_security_group_ids= [app_security_group.id],
    ami="ami-05f4d8898209c4f55",  # Amazon Linux 2 AMI ID; update if needed
    user_data=user_data_script,
    key_name=ecs_key_pair.key_name,
    tags={
        "Name": "MySQL-Server",
    }
)

# # Target Group Attachments
# ubuntu_target_attachment = aws.lb.TargetGroupAttachment("ubuntu-tg-attachment",
#     target_group_arn=target_group.arn,
#     target_id=ubuntu_instance.id,
#     port=80
# )

# amazon_linux_target_attachment = aws.lb.TargetGroupAttachment("amazon-linux-tg-attachment",
#     target_group_arn=target_group.arn,
#     target_id=amazon_linux_instance.id,
#     port=80
# )

ecs_cluster = aws.ecs.Cluster("DevCluster",  # Changed name to match image
    tags={"Name": "DevCluster"}
)

# IAM Role for ECS instances
# ecs_instance_role = aws.iam.Role("ecs-instance-role",
#     assume_role_policy=json.dumps({
#         "Version": "2012-10-17",
#         "Statement": [{
#             "Action": "sts:AssumeRole",
#             "Principal": {
#                 "Service": "ec2.amazonaws.com"
#             },
#             "Effect": "Allow",
#             "Sid": ""
#         }]
#     })
# )

# # Attach AmazonEC2ContainerServiceforEC2Role policy to the role
# ecs_instance_role_policy_attachment = aws.iam.RolePolicyAttachment("ecs-instance-role-policy",
#     role=ecs_instance_role.name,
#     policy_arn="arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
# )

# # Create instance profile for ECS instances
# ecs_instance_profile = aws.iam.InstanceProfile("ecs-instance-profile",
#     role=ecs_instance_role.name
# )

# Create a key pair for ECS instances


ecs_launch_template = aws.ec2.LaunchTemplate("ecs-launch-template",
    name_prefix="ecs-template",
    image_id="ami-04ef3193cda2ac9d4",
    instance_type="t2.micro",
    key_name=ecs_key_pair.key_name,
    vpc_security_group_ids=[app_security_group.id],
    block_device_mappings=[{
        "deviceName": "/dev/xvda",
        "ebs": {
            "volumeSize": 30,
            "volumeType": "gp2",
            "deleteOnTermination": True
        }
    }],
    user_data=(
        base64.b64encode(f"""#!/bin/bash
echo ECS_CLUSTER={ecs_cluster.name} >> /etc/ecs/ecs.config
""".encode("ascii")).decode("ascii")
    ),
    tags={"Name": "ecs-launch-template"},
    opts=pulumi.ResourceOptions(depends_on=[ecs_key_pair])  # Add this line
)

# Auto Scaling Group for ECS instances
ecs_asg = aws.autoscaling.Group("ecs-asg",
    vpc_zone_identifiers=[
        public_subnet_1.id,
        public_subnet_2.id,
        public_subnet_3.id
    ],
    desired_capacity=1,
    min_size=0,  # Changed to 0 as shown in image
    max_size=5,  # Kept at 5 as shown in image
    launch_template={
        "id": ecs_launch_template.id,
        "version": "$Latest"
    },
    tags=[
        {
            "key": "Name",
            "value": "ecs-instance",
            "propagate_at_launch": True
        },
        {
            "key": "AmazonECSManaged",
            "value": "",
            "propagate_at_launch": True
        }
    ]
)

# Capacity Provider
ecs_capacity_provider = aws.ecs.CapacityProvider("dev-capacity-provider",  # Changed from "ecs-capacity-provider"
    auto_scaling_group_provider={
        "autoScalingGroupArn": ecs_asg.arn,
        "managedScaling": {
            "status": "ENABLED",
            "targetCapacity": 100,
            "minimumScalingStepSize": 1,
            "maximumScalingStepSize": 1,
            "instanceWarmupPeriod": 300
        },
        "managedTerminationProtection": "DISABLED"
    },
    tags={"Name": "dev-capacity-provider"}  # Update tag to match new name
)

# Update the cluster capacity providers reference as well
ecs_cluster_capacity_providers = aws.ecs.ClusterCapacityProviders("cluster-capacity-providers",  # Changed from "ecs-cluster-capacity-providers"
    cluster_name=ecs_cluster.name,
    capacity_providers=[ecs_capacity_provider.name],
    default_capacity_provider_strategies=[{
        "base": 1,
        "weight": 100,
        "capacityProvider": ecs_capacity_provider.name
    }]
)

# Task Definition
ecs_task_definition = aws.ecs.TaskDefinition("app-task",
    family="app-task",
    requires_compatibilities=["EC2"],
    cpu="256",
    memory="512",
    network_mode="awsvpc",
    execution_role_arn="arn:aws:iam::381492238196:role/ecsTaskExecutionRole",  # Ensure your execution role is set up
    container_definitions=json.dumps([{
        "name": "app-container",
        "image": "poridhi/laravel:v3.1",
        "essential": True,
        "portMappings": [{
            "containerPort": 8000,
            "hostPort": 8000,
            "protocol": "tcp"
        }],
         "environment": [
                {
                    "name": "APP_ENV",
                    "value": "local"
                },
                {
                    "name": "DB_USERNAME",
                    "value": "poridhi"
                },
                {
                    "name": "DB_PORT",
                    "value": "3306"
                },
                {
                    "name": "DB_CONNECTION",
                    "value": "mysql"
                },
                {
                    "name": "APP_NAME",
                    "value": "Laravel"
                },
                {
                    "name": "DB_HOST",
                    "value": "13.208.184.196"
                },
                
                {
                    "name": "DB_DATABASE",
                    "value": "poridhi"
                },
                {
                    "name": "APP_DEBUG",
                    "value": "true"
                },
                {
                    "name": "DB_PASSWORD",
                    "value": "Poridhi@1809014"
                }
            ]
    }]),
    tags={"Name": "app-task"}
)

# ECS Service
ecs_service = aws.ecs.Service("app-service",
    cluster=ecs_cluster.arn,
    task_definition=ecs_task_definition.arn,
    desired_count=1,
    launch_type="EC2",
    network_configuration=aws.ecs.ServiceNetworkConfigurationArgs(
        subnets=[private_subnet_1.id, private_subnet_2.id],
        security_groups=[app_security_group.id,alb_security_group.id],
        assign_public_ip=False,
    ),
    load_balancers=[aws.ecs.ServiceLoadBalancerArgs(
        target_group_arn=target_group.arn,
        container_name="app-container",
        container_port=8000
    )],
    tags={"Name": "app-service"}
)

# Export the ECS task definition name
pulumi.export("ecs_task_definition_name", ecs_task_definition.family)
pulumi.export("ecs_service_name", ecs_service.name)


# Export important values
pulumi.export('ecs_cluster_name', ecs_cluster.name)
pulumi.export('ecs_key_pair_name', ecs_key_pair.key_name)

# Export important values
pulumi.export('vpc_id', vpc.id)
pulumi.export('alb_dns_name', alb.dns_name)
pulumi.export('ubuntu_instance_ip', ubuntu_instance.public_ip)
pulumi.export('amazon_linux_instance_ip', amazon_linux_instance.public_ip)
```

### 7. Deployment
```bash
# Initialize Pulumi project
pulumi stack init dev

# Preview changes
pulumi preview

# Deploy infrastructure
pulumi up
```

## Post-Deployment Steps

### 1. Database Setup
1. SSH into the MySQL instance:
   ```bash
   ssh -i ~/.ssh/ecs-key ubuntu@<mysql-instance-ip>
   ```
2. Configure MySQL:
   ```bash
   sudo mysql
   CREATE DATABASE poridhi;
   CREATE USER 'poridhi'@'%' IDENTIFIED BY 'Poridhi@1809014';
   GRANT ALL PRIVILEGES ON poridhi.* TO 'poridhi'@'%';
   FLUSH PRIVILEGES;
   ```

### 2. Verify Deployment
1. Check ECS cluster:
   - Access AWS ECS console
   - Verify cluster `DevCluster` is running
   - Check service `app-service` is active

2. Verify Application:
   - Access the application through ALB DNS:
     ```
     http://<alb-dns-name>:8000
     ```

## Infrastructure Details

### Network Architecture
- VPC CIDR: 10.0.0.0/16
- Public Subnets:
  - 10.0.1.0/24 (ap-northeast-3a)
  - 10.0.2.0/24 (ap-northeast-3b)
  - 10.0.3.0/24 (ap-northeast-3c)
- Private Subnets:
  - 10.0.4.0/24 (ap-northeast-3a)
  - 10.0.5.0/24 (ap-northeast-3b)

### Security Groups
1. Application Security Group (laravel-sg):
   - Inbound: 22, 80, 8000, 3306, 2049
   - Outbound: All traffic

2. ALB Security Group (alb-sg):
   - Inbound: 80, 443
   - Outbound: All traffic

### ECS Configuration
- Cluster: DevCluster
- Instance Type: t2.micro
- Auto Scaling:
  - Min: 0
  - Desired: 1
  - Max: 5

## Cleanup
To destroy the infrastructure:
```bash
pulumi destroy
```

## Troubleshooting

### Common Issues and Solutions

1. ECS Service Not Starting
   - Check ECS service events in AWS console
   - Verify security group rules
   - Check task definition environment variables

2. Database Connectivity Issues
   - Verify MySQL instance is running
   - Check security group rules
   - Confirm database credentials in task definition

3. Load Balancer Health Checks Failing
   - Verify target group configuration
   - Check security group rules
   - Confirm application is running on correct port

