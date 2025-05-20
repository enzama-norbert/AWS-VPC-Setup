AWS VPC SETUP: ENABLE PRIVATE SUBNET INTERNET ACCESS VIA NAT GATEWAY

A secure, scalable architecture for private EC2 instances requiring outbound internet access

📌 Overview
This guide walks through configuring an AWS VPC with public and private subnets, allowing resources in private subnets (e.g., databases, internal services) to securely access the internet via a NAT Gateway, while blocking unsolicited inbound traffic.

Architecture Diagram
AWS VPC with NAT Gateway (Replace with your diagram)
![AWS VPC Setup - Enable Private Subnet Internet Access via NAT Gateway](https://github.com/user-attachments/assets/c38c1dd1-a27e-494a-bcf8-97b3f746202d)


🛠️ Prerequisites
An AWS account with IAM permissions to create VPCs, subnets, and EC2 instances.

Basic familiarity with AWS CLI or Management Console.

🚀 Step-by-Step Deployment/ Run these commands to deploy the infrastructure: 
1. Create a VPC
<img width="959" alt="Create a VPC" src="https://github.com/user-attachments/assets/ce599949-7ca8-455e-b6f2-300399727ea3" />
  
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=NAT-Demo}]'
```

2. Create Subnets

**Public Subnet (10.0.1.0/24):**

<img width="955" alt="manara-demo-public1" src="https://github.com/user-attachments/assets/348e062c-847e-412a-bd4f-b05fe32eae3a" />

```bash
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
```

**Private Subnet (10.0.2.0/24):**

<img width="947" alt="manara-demo-private1" src="https://github.com/user-attachments/assets/6b20b99d-ff3b-438f-8d34-e7675fa6d7b5" />

```bash
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.2.0/24 --availability-zone us-east-1a
```

3. Create and Attach an Internet Gateway (IGW)

   <img width="949" alt="manara-demo-IGW" src="https://github.com/user-attachments/assets/024f0edd-16d0-4d9e-8d09-b5532e9adfe2" />

```bash
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Demo-IGW}]'
aws ec2 attach-internet-gateway --internet-gateway-id <IGW_ID> --vpc-id <VPC_ID>
```

4. Deploy a NAT Gateway
Allocate an Elastic IP:

<img width="938" alt="manara-demo-NAT-Elastic IP" src="https://github.com/user-attachments/assets/e8d95e5a-dd6c-4d9c-93a3-c404fe52a0d1" />
```bash
aws ec2 allocate-address --domain vpc
```

Create NAT Gateway in a public subnet:

<img width="950" alt="manara-demo-NAT Gateway" src="https://github.com/user-attachments/assets/473e78e8-ddf3-4d8c-8d85-cdb6db6d80a5" />
```bash
aws ec2 create-nat-gateway --subnet-id <PUBLIC_SUBNET_ID> --allocation-id <EIP_ALLOC_ID> --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=Demo-NAT}]'
```

5. Configure Route Tables
Public Route Table: Route 0.0.0.0/0 → IGW

<img width="950" alt="manara-demo-rt-public" src="https://github.com/user-attachments/assets/7fcd6f44-6e3c-4bdd-bc56-72e66be3348a" />
```bash
aws ec2 create-route-table --vpc-id <VPC_ID> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]'
aws ec2 create-route --route-table-id <PUBLIC_RT_ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW_ID>
aws ec2 associate-route-table --route-table-id <PUBLIC_RT_ID> --subnet-id <PUBLIC_SUBNET_ID>
```

Private Route Table: Route 0.0.0.0/0 → NAT Gateway

<img width="956" alt="manara-demo-rt-private" src="https://github.com/user-attachments/assets/7b61da99-c0af-44c8-ae0c-e1164a3b7eb2" />
```bash
aws ec2 create-route-table --vpc-id <VPC_ID> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-RT}]'
aws ec2 create-route --route-table-id <PRIVATE_RT_ID> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <NAT_ID>
aws ec2 associate-route-table --route-table-id <PRIVATE_RT_ID> --subnet-id <PRIVATE_SUBNET_ID>
```

6. Launch Test Instances
Public Instance (Bastion Host):

<img width="944" alt="webserver" src="https://github.com/user-attachments/assets/61475462-1568-471e-adc9-c36220cfabb9" />

```bash
aws ec2 run-instances --image-id ami-0abcdef1234567890 --count 1 --instance-type t2.micro --subnet-id <PUBLIC_SUBNET_ID> --security-group-ids <SG_ID> --key-name <KEY_PAIR> --associate-public-ip-address --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Bastion}]'
```
Private Instance (No Public IP):

```bash
aws ec2 run-instances --image-id ami-0abcdef1234567890 --count 1 --instance-type t2.micro --subnet-id <PRIVATE_SUBNET_ID> --security-group-ids <SG_ID> --key-name <KEY_PAIR> --no-associate-public-ip-address --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Private-Server}]'
```
🔐 Security Best Practices
Restrict Inbound Traffic: Use Security Groups and NACLs to limit access to private instances.

Use SSM Session Manager: Avoid exposing bastion hosts by leveraging AWS Systems Manager.

VPC Endpoints: Reduce NAT costs by using endpoints for S3/DynamoDB.

Tag Everything: Track resources for cost allocation and cleanup.

📦 Cleanup
To avoid unnecessary charges, delete resources after testing:

```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
aws ec2 delete-nat-gateway --nat-gateway-id <NAT_ID>
aws ec2 release-address --allocation-id <EIP_ALLOC_ID>
aws ec2 delete-vpc --vpc-id <VPC_ID>
```
⁉️ FAQ
Q: Why not use a NAT Instance instead?
A: NAT Gateway is fully managed, scales automatically, and avoids single-point failures.

Q: How do I access private instances?
A: Use AWS Systems Manager Session Manager or a bastion host in the public subnet.

📚 References
AWS NAT Gateway Documentation

VPC Design Best Practices

💬 Feedback Welcome!
Open an issue or PR if you have suggestions or improvements.
