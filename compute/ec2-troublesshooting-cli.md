# Amazon EC2: CLI Automation & Troubleshooting (LAMP Stack)

## 1. Overview & Key Features

Deploying infrastructure via the AWS CLI allows for powerful automation, but it also introduces opportunities for misconfigurations. This guide covers how to deploy a full LAMP (Linux, Apache, MySQL/MariaDB, PHP) stack using AWS CLI shell scripts and User Data, alongside techniques for troubleshooting common provisioning and networking issues.

**Key Features Explored:**

* **LAMP Stack Automation:** Deploying a multi-tier web application using a single EC2 instance and a `user-data` bootstrapping script.
* **CLI Scripting:** Using Bash scripts to dynamically query VPCs, Subnets, and AMIs, and pass those variables into the `aws ec2 run-instances` command.
* **Diagnostic Tools:** Using network mappers (`nmap`) and system logs (`cloud-init`) to identify why an instance is failing to launch or serve traffic.

## 2. Core Concepts

* **LAMP Stack:** A common open-source software bundle used to host web applications. It stands for **L**inux (OS), **A**pache (Web Server), **M**ySQL/MariaDB (Database), and **P**HP (Programming Language).
* **User Data / Cloud-Init:** A script passed to an EC2 instance at launch. The `cloud-init` service runs this script automatically during the first boot cycle to install software and configure the OS.
* **AMI Regionality:** Amazon Machine Images (AMIs) are Region-specific. An AMI ID that exists in `us-east-1` will not exist in `us-west-2`.
* **Nmap:** A popular open-source network scanning tool used to discover hosts and services on a computer network by sending packets and analyzing the responses.

## 3. Hands-On Guide: Deploying and Troubleshooting

### Task A: Prepare the Environment & Script Backup

When working with automation scripts, always create a backup before making modifications.

1. Connect to your terminal (or CLI Host instance).
2. Configure your environment using `aws configure`.
3. Backup your deployment script:
```bash
cp create-lamp-instance.sh create-lamp-instance.backup

```



### Task B: Analyze the Deployment Script

A robust AWS CLI deployment script typically follows these steps:

1. **Query Infrastructure:** Uses `aws ec2 describe-regions` and `describe-subnets` to dynamically find the correct VPC and Subnet IDs.
2. **Clean Up:** Checks for existing resources (like old instances or security groups) and deletes them to prevent conflicts.
3. **Provision Security:** Uses `aws ec2 create-security-group` to create a firewall allowing ports 22 (SSH) and 80 (HTTP).
4. **Launch:** Executes `aws ec2 run-instances`, passing the dynamically gathered variables and the `--user-data` file.

### Task C: Troubleshooting Issue #1 - AMI Not Found

**Symptom:** Running the script results in `An error occurred (InvalidAMIID.NotFound) when calling the RunInstances operation`.
**Diagnosis:**

* AMIs are region-specific. If the script queries an AMI in one region but attempts to launch the instance in another without specifying the `--region` flag, AWS will not find the AMI.
**Resolution:**
* Ensure the `aws ec2 run-instances` command explicitly includes the correct `--region` parameter that matches the AMI's location.

### Task D: Troubleshooting Issue #2 - Service Unreachable

**Symptom:** The instance launches and receives a Public IP, but the website times out or refuses to connect.
**Diagnosis Steps:**

1. **Check the Network (Port Scanning):**
Install `nmap` and scan the public IP to see if port 80 is actively listening.
```bash
sudo yum install -y nmap
nmap -Pn <public-ip>

```


*If port 80 is closed, the issue is either the Security Group blocking traffic, or the Web Server service (`httpd`) is not running.*
2. **Check the Bootstrapping Logs:**
Connect to the instance via SSH or EC2 Instance Connect and verify if the User Data script executed successfully.
```bash
sudo tail -f /var/log/cloud-init-output.log

```


*Look for errors in package installations (Apache/PHP/MariaDB) or syntax errors in the script that caused the web server to fail its startup sequence.*

## 4. Diagnostics & Code Snippets Quick Reference

**View the full Cloud-Init Execution Log:**
Whenever a User Data script fails, this is the first file you should check.

```bash
sudo cat /var/log/cloud-init-output.log

```

**Port Scanning a Remote Instance:**
Useful to verify if Security Groups are actually allowing the traffic you expect.

```bash
nmap -Pn <instance-public-ip>

```

**Example: Dynamic Subnet Query in Bash:**
How to programmatically find a subnet ID based on a Tag Name using the AWS CLI.

```bash
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public Subnet" --query "Subnets[0].SubnetId" --output text)

```

## 5. Best Practices & Security

* **Idempotent Scripts:** Design your deployment scripts to be idempotent (they can be run multiple times safely). Include cleanup logic or checks to see if a resource already exists before trying to create it.
* **Protect User Data:** User Data scripts are visible to anyone with `ec2:DescribeInstances` API permissions. Never hardcode sensitive database passwords or API keys in plain text within User Data. Use AWS Systems Manager Parameter Store or AWS Secrets Manager instead.
* **Security Group Scoping:** While port 80 (HTTP) usually needs to be open to the world (`0.0.0.0/0`), Port 22 (SSH) should *always* be restricted to your specific IP address or a Bastion Host Security Group.

---