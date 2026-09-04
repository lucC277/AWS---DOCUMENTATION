# Amazon EC2: Advanced Launch & Bastion Hosts (Console + CLI)

## 1. Overview & Key Features

While the AWS Management Console is great for quickly launching temporary instances, the AWS Command Line Interface (CLI) allows you to automate the provisioning of AWS resources in a repeatable and reliable manner.

This guide covers how to deploy a **Bastion Host** via the console and use it to securely launch and manage a private/internal Web Server using the AWS CLI.

**Key Features Explored:**

* **Bastion Hosts:** Secure gateways to access internal/private instances.
* **EC2 Instance Connect:** A simple and secure way to connect to your instances using your browser without managing SSH keys.
* **CLI Automation:** Programmatically query AWS for dynamic variables (like the latest AMI IDs) and launch resources.
* **Instance Metadata:** Retrieve real-time data about an instance from within the instance itself.

## 2. Core Concepts

* **Bastion Host (Jump Box):** A special-purpose server configured to withstand attacks, used as a proxy to access instances in private subnets from an external network.
* **EC2 Instance Connect:** Provides a browser-based, one-click secure SSH connection to your Linux instances.
* **IAM Instance Profile:** An IAM Role attached directly to an EC2 instance, allowing the applications (or CLI) running on the instance to securely make API requests to AWS services without hardcoding credentials.
* **Instance Metadata Service (IMDS):** A local endpoint (`[http://169.254.169.254/latest/meta-data/](http://169.254.169.254/latest/meta-data/)`) accessible from within the EC2 instance to query its own configuration details (e.g., Availability Zone, IP address).

## 3. Hands-On Guide: Architecture Deployment

### Task A: Launch a Bastion Host (AWS Console)

1. Go to the **EC2 Dashboard** and click **Launch instance**.
2. **Name:** `Bastion host`
3. **AMI:** Amazon Linux 2
4. **Instance Type:** `t3.micro`
5. **Key Pair:** Select **Proceed without a key pair** (since we will use EC2 Instance Connect).
6. **Network Settings:** Select your VPC and a **Public Subnet**. Set Auto-assign public IP to **Enable**.
7. **Security Group:** Create a new one named `Bastion security group` allowing inbound SSH (Port 22).
8. **Advanced Details:** Attach an **IAM Instance Profile** (e.g., `Bastion-Role`) so the instance has permissions to run AWS CLI commands.
9. **Launch** the instance.

### Task B: Connect Securely without Keys

1. Select the `Bastion host` instance in the EC2 console.
2. Click **Connect**.
3. Choose the **EC2 Instance Connect** tab and click **Connect**. A browser-based SSH terminal will open securely.

### Task C: Launch a Web Server via AWS CLI

From inside the Bastion Host terminal, run the following automated deployment:

**1. Retrieve Dynamic Variables:**
Find the Availability Zone, query Systems Manager (SSM) for the latest Amazon Linux 2 AMI, and find the Subnet and Security Group IDs.

```bash
# Get current Region from Instance Metadata
AZ=`curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone`
export AWS_DEFAULT_REGION=${AZ::-1}

# Retrieve latest Linux AMI ID from SSM Parameter Store
AMI=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --query 'Parameters[0].[Value]' --output text)

# Retrieve Subnet ID
SUBNET=$(aws ec2 describe-subnets --filters 'Name=tag:Name,Values=Public Subnet' --query Subnets[].SubnetId --output text)

# Retrieve Security Group ID
SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=WebSecurityGroup --query SecurityGroups[].GroupId --output text)

```

**2. Download User Data Script:**

```bash
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-1-23732/171-lab-JAWS-create-ec2/s3/UserData.txt

```

**3. Launch the Instance:**

```bash
INSTANCE=$(aws ec2 run-instances \
  --image-id $AMI \
  --subnet-id $SUBNET \
  --security-group-ids $SG \
  --user-data file:///home/ec2-user/UserData.txt \
  --instance-type t3.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Web Server}]' \
  --query 'Instances[*].InstanceId' \
  --output text)

```

**4. Monitor & Test:**
Wait for the instance state to change to `running`, then grab its Public DNS to access the web server.

```bash
# Check status
aws ec2 describe-instances --instance-ids $INSTANCE --query 'Reservations[].Instances[].State.Name' --output text

# Get Public DNS to test in browser
aws ec2 describe-instances --instance-ids $INSTANCE --query Reservations[].Instances[].PublicDnsName --output text

```

## 4. Best Practices & Security

* **Provisioning Strategy:**
* *AWS Console:* Best for one-off tasks, testing, and quick prototyping.
* *AWS CLI / Scripts:* Best for automated, repeatable, and reliable provisioning.
* *CloudFormation / Terraform:* Best for deploying related resources together (Infrastructure as Code).


* **Bastion Security:** Never store private SSH keys on a Bastion Host. Use SSH Agent Forwarding or EC2 Instance Connect to securely pass credentials through.
* **Dynamic AMIs:** Hardcoding AMI IDs in scripts is dangerous because AMIs are updated and deprecated frequently. Always query a service like **SSM Parameter Store** to fetch the latest patched AMI ID dynamically.

---