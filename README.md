# Launching an EC2 instance in a Custom Virtual Private Cloud (VPC)
#aws/certified/cloud-practitioner #aws/ec2 #aws/aws-cli

Adapted from: [A Cloud Guru | Launching an EC2 instance in a Custom Virtual Private Cloud (VPC)](https://learn.acloud.guru/handson/7e6a058b-92d1-468b-8ad0-b499029e5337/course/aws-certified-cloud-practitioner)

## Overview
![](https://i.ibb.co/QQSgtLB/AF43-CCB6-C1-D8-49-AB-BF53-2702-A6-D11577.png)

```
# Create custom VPC
# Create public and private subnets
# Create routes and configure internet gateway
# Launch EC2 instances in subnet
# Access EC2 instances
```

## VPC
[Create a custom VPC and return VPC ID](https://cloudaffaire.com/how-to-create-a-custom-vpc-using-aws-cli/#:~:text=Step%202%3A%20Create%20a%20VPC.):
```bash
# Create a VPC that's isolated from other private networks
# in order to secure your resources
AWS_VPC_ID=$(aws ec2 create-vpc \
--cidr-block 10.0.0.0/16 \
--query 'Vpc.VpcId' \
--output text) && echo $AWS_VPC_ID
```

`—cider-block` - The IPv4 network range for the VPC, in CIDR notation.

Check VPC created:
```bash
aws ec2 describe-vpcs \
--query "Vpcs[?VpcId=='$AWS_VPC_ID'].VpcId" \
--output text
```

Create name tag for VPC:
```bash
aws ec2 create-tags \
--resources $AWS_VPC_ID \
--tags Key=Name,Value=HoLVPC
```

Check tag created:
```bash
aws ec2 describe-tags \
--query "Tags[?Value=='HoLVPC'].Value" \
--output text
```

## Subnets
Create public and private subnets:
```bash
# Resources and services are launched in subnets not the VPC
# e.g. EC2 instances are launched in subnets not VPCs

# This won't make the subnet public over the internet just yet
AWS_SUBNET_PUBLIC_ID=$(aws ec2 create-subnet \
--vpc-id $AWS_VPC_ID \
--availability-zone us-east-1a \
--cidr-block 10.0.1.0/24 \
--query "Subnet.SubnetId" \
--output text) && echo $AWS_SUBNET_PUBLIC_ID

AWS_SUBNET_PRIVATE_ID=$(aws ec2 create-subnet \
--vpc-id $AWS_VPC_ID \
--availability-zone us-east-1b \
--cidr-block 10.0.2.0/24 \
--query "Subnet.SubnetId" \
--output text) && echo $AWS_SUBNET_PRIVATE_ID
```

Create tags for subnets
```bash
aws ec2 create-tags \
--resources $AWS_SUBNET_PUBLIC_ID \
--tags Key=Name,Value=sn-public-a \
&& \
aws ec2 create-tags \
--resources $AWS_SUBNET_PRIVATE_ID \
--tags Key=Name,Value=sn-private-b
```

Check subnets created:
```bash
aws ec2 describe-subnets \
--query \
"Subnets[?contains(['$AWS_SUBNET_PUBLIC_ID', '$AWS_SUBNET_PRIVATE_ID'], SubnetId)]"
```

## Internet Gateway
Make public subnets public over internet:
```bash
aws ec2 modify-subnet-attribute \
--subnet-id $AWS_SUBNET_PUBLIC_ID \
--map-public-ip-on-launch

# check it's on
aws ec2 describe-subnets \
--query \
"Subnets[?SubnetId=='$AWS_SUBNET_PUBLIC_ID'].MapPublicIpOnLaunch" \
--output text
```

`—map-public-ip-on-launch` - enables auto-assign ipv4

Create internet gateway:
```bash
# Routes inside a VPC defines the movement of traffic within subnets and to/from the internet
# The internet gateway will allow the VPC to connect to the internet
AWS_INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
--query "InternetGateway.InternetGatewayId" \
--output text) \
&& echo $AWS_INTERNET_GATEWAY_ID

# attach name tag
aws ec2 create-tags \
--resources $AWS_INTERNET_GATEWAY_ID \
--tags Key=Name,Value=HoLVPC-IG
```

Attach internet gateway to VPC:
```bash
aws ec2 attach-internet-gateway \
--vpc-id $AWS_VPC_ID \
--internet-gateway-id $AWS_INTERNET_GATEWAY_ID
```

## Route
Create route table:
```bash
AWS_ROUTE_TABLE_ID=$(\
aws ec2 create-route-table \
--vpc-id $AWS_VPC_ID \
--query "RouteTable.RouteTableId" \
--output text) \
&& echo $AWS_ROUTE_TABLE_ID

# attach name tag
aws ec2 create-tags \
--resources $AWS_ROUTE_TABLE_ID \
--tags Key=Name,Value=publicRT
```

Allow route to access internet:
```bash
aws ec2 create-route \
--route-table-id $AWS_ROUTE_TABLE_ID \
--destination-cidr-block 0.0.0.0/0 \
--gateway-id $AWS_INTERNET_GATEWAY_ID

# check
aws ec2 describe-route-tables \
--query "RouteTables[?RouteTableId=='$AWS_ROUTE_TABLE_ID']"
```

Direct traffic in the public subnet by associating subnet with route table:
```bash
AWS_ROUTE_TABLE_ASSOCIATE_ID=$(\
aws ec2 associate-route-table \
--subnet-id $AWS_SUBNET_PUBLIC_ID \
--route-table-id $AWS_ROUTE_TABLE_ID \
--output text) \
&& echo $AWS_ROUTE_TABLE_ASSOCIATE_ID
```

## EC2
[View list of all Linux AMIs in current region](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html#:~:text=You%20can%20view%20a%20list%20of%20all%20Linux%20AMIs%20in%20the%20current%20AWS%20Region%20by%20using%20the%20following%20command%20in%20the%20AWS%20CLI):
```bash
aws ssm get-parameters-by-path \
--path /aws/service/ami-amazon-linux-latest \
--query "Parameters[].Name"

# Set variable for latest Amazon Linux 2 AMI
AWS_IMAGE_PATH=/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
&& echo $AWS_IMAGE_PATH
```

[Create security groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-security-group.html):
```bash
AWS_PUBLIC_SECURITY_GROUP_ID=$(aws ec2 create-security-group \
--group-name HoLPubSG \
--description HoLPubSG \
--vpc-id $AWS_VPC_ID \
--query "GroupId" \
--output text) \
&& echo $AWS_PUBLIC_SECURITY_GROUP_ID

AWS_PRIVATE_SECURITY_GROUP_ID=$(aws ec2 create-security-group \
--group-name HoLPrivSG \
--description HoLPrivSG \
--vpc-id $AWS_VPC_ID \
--query "GroupId" \
--output text) \
&& echo $AWS_PRIVATE_SECURITY_GROUP_ID
```

[Add security group rules](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-ingress.html):
```bash
# Public security group rule
aws ec2 authorize-security-group-ingress \
--group-id $AWS_PUBLIC_SECURITY_GROUP_ID \
--protocol tcp \
--port 22 \
--cidr 0.0.0.0/0 # not recommended in real world

# Private security group rule
# Allow the IP of the public subnet to access it via SSH
aws ec2 authorize-security-group-ingress \
--group-id $AWS_PRIVATE_SECURITY_GROUP_ID \
--protocol tcp \
--port 22 \
--cidr 10.0.1.0/24
```

[Create key pairs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-key-pair.html):
```bash
AWS_PUBLIC_KEY_NAME=HoLPubKeyPair \
&& \
aws ec2 create-key-pair \
--key-name $AWS_PUBLIC_KEY_NAME \
--query "KeyMaterial" \
--output text > $AWS_PUBLIC_KEY_NAME.pem

AWS_PRIVATE_KEY_NAME=HoLPrivKeyPair \
&& \
aws ec2 create-key-pair \
--key-name $AWS_PRIVATE_KEY_NAME \
--query "KeyMaterial" \
--output text > $AWS_PRIVATE_KEY_NAME.pem
```

[Launch EC2 instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html#:~:text=query%20%22Parameters%5B%5D.Name%22-,To%20launch%20an%20instance%20using%20a%20public%20parameter,-The%20following%20example):
```bash
# Using the latest Amazon Linux 2 AMI

# Launch in public subnet
AWS_PUBLIC_EC2_INSTANCE_ID=$(aws ec2 run-instances \
--image-id resolve:ssm:$AWS_IMAGE_PATH \
--instance-type t2.micro \
--count 1 \
--subnet-id $AWS_SUBNET_PUBLIC_ID \
--associate-public-ip-address \
--security-group-ids $AWS_PUBLIC_SECURITY_GROUP_ID \
--key-name $AWS_PUBLIC_KEY_NAME \
--tag-specifications \
'ResourceType=instance,Tags=[{Key=Name,Value=ec2-public}]' \
--query "Instances[].InstanceId" \
--output text) \
&& echo $AWS_PUBLIC_EC2_INSTANCE_ID

# Launch in private subnet
AWS_PRIVATE_EC2_INSTANCE_ID=$(aws ec2 run-instances \
--image-id resolve:ssm:$AWS_IMAGE_PATH \
--instance-type t2.micro \
--count 1 \
--subnet-id $AWS_SUBNET_PRIVATE_ID \
--no-associate-public-ip-address \
--security-group-ids $AWS_PRIVATE_SECURITY_GROUP_ID \
--key-name $AWS_PRIVATE_KEY_NAME \
--tag-specifications \
'ResourceType=instance,Tags=[{Key=Name,Value=ec2-private}]' \
--query "Instances[].InstanceId" \
--output text) \
&& echo $AWS_PRIVATE_EC2_INSTANCE_ID
```

Get public IP address for public instance:
```bash
AWS_PUBLIC_EC2_INSTANCE_IP=$(aws ec2 describe-instances \
--query \
"Reservations[].Instances[?InstanceId=='$AWS_PUBLIC_EC2_INSTANCE_ID'].PublicIpAddress" \
--output text) \
&& echo $AWS_PUBLIC_EC2_INSTANCE_IP
```

Connect to EC2 instances:
```bash
# Run this chmod command to ensure key is not publicly viewable:
chmod 400 $AWS_PUBLIC_KEY_NAME.pem

# Connect to instance
ssh -i "$AWS_PUBLIC_KEY_NAME.pem" ec2-user@$AWS_PUBLIC_EC2_INSTANCE_IP
```

Create private key pair inside instance:
```bash
nano HoLPrivKeyPair.pem
# copy and paste content from key pair file into this one
```

Connect to private EC2 instance inside the public EC2 instance.
