Aqui está a documentação do **Amazon VPC**,
---

# Amazon Virtual Private Cloud (VPC)

## 1. Overview & Key Features

Amazon Virtual Private Cloud (Amazon VPC) enables you to launch AWS resources into a virtual network that you've defined. This virtual network closely resembles a traditional network that you'd operate in your own data center, with the benefits of using the scalable infrastructure of AWS.

**Key Features:**

* **Custom Network Configuration:** Full control over your virtual networking environment, including resource placement, connectivity, and security.
* **Subnetting:** Divide your network into public and private tiers across multiple Availability Zones (AZs).
* **Advanced Routing:** Use route tables to control where network traffic is directed.
* **Security Controls:** Multiple layers of security, including Security Groups (instance level) and Network Access Control Lists (NACLs, subnet level).

## 2. Core Concepts

* **CIDR Block (Classless Inter-Domain Routing):** The IP address range assigned to your VPC (e.g., `10.0.0.0/16`) and its subnets (e.g., `10.0.1.0/24`).
* **Subnet:** A range of IP addresses in your VPC.
* **Public Subnet:** Has a direct route to the Internet Gateway.
* **Private Subnet:** Does not have a direct route to the Internet Gateway.


* **Internet Gateway (IGW):** A horizontally scaled, redundant VPC component that allows communication between instances in your VPC and the internet.
* **NAT Gateway:** Enables instances in a private subnet to connect to the internet (for updates/patches) but prevents the internet from initiating connections with those instances.
* **Route Table:** A set of rules, called routes, that are used to determine where network traffic is directed.

## 3. Hands-On Guide: Building a Custom VPC Architecture

### Task A: Create the Foundation (VPC & Initial Subnets)

1. Go to the **VPC Dashboard** and choose **Create VPC**.
2. Select **VPC and more** to generate multiple resources at once.
3. Configure the **IPv4 CIDR block** (e.g., `10.0.0.0/16`).
4. Set the number of **Availability Zones** to 1 (initially).
5. Specify **1 Public Subnet** and **1 Private Subnet**, defining their respective CIDR blocks (e.g., `10.0.0.0/24` and `10.0.1.0/24`).
6. Enable a **NAT Gateway** in 1 AZ.
7. Click **Create VPC** to automatically generate the VPC, Subnets, Route Tables, Internet Gateway, and NAT Gateway.

### Task B: Expand for High Availability (Adding Subnets)

To ensure High Availability, you should span your network across multiple Availability Zones.

1. Navigate to **Subnets** > **Create subnet**.
2. Select your VPC.
3. Create a **second Public Subnet** (e.g., `10.0.2.0/24`) and a **second Private Subnet** (e.g., `10.0.3.0/24`).
4. Assign these new subnets to a different Availability Zone (e.g., if the first ones were in AZ-a, place these in AZ-b).

### Task C: Associate Subnets with Route Tables

Newly created subnets need to be explicitly associated with the correct route tables to function as public or private.

1. Navigate to **Route Tables**.
2. Select the **Public Route Table**, go to the **Subnet associations** tab, and edit associations. Check the box for your newly created Public Subnet.
3. Select the **Private Route Table**, go to the **Subnet associations** tab, and check the box for your newly created Private Subnet.

### Task D: Create a VPC Security Group

1. Navigate to **Security Groups** > **Create security group**.
2. Name it (e.g., `Web Security Group`) and ensure your custom VPC is selected.
3. Add an **Inbound rule** to allow HTTP traffic (Type: `HTTP`, Source: `Anywhere-IPv4`).

### Task E: Launch a Web Server into the VPC

1. Go to the **EC2 Dashboard** > **Launch instances**.
2. Choose an AMI (e.g., Amazon Linux 2) and Instance Type (e.g., `t3.micro`).
3. Under **Network settings**, explicitly select your custom VPC and one of your **Public Subnets**.
4. Set **Auto-assign public IP** to **Enable**.
5. Select your existing **Web Security Group**.
6. Provide a **User Data** script to install the web server software (see below).
7. Launch the instance and test accessibility using its Public IPv4 DNS.

## 4. Code Snippets

**User Data Script (LAMP Stack setup & App Deployment):**
This script installs Apache and PHP, downloads a sample application from an S3 bucket, unzips it into the web directory, and starts the server.

```bash
#!/bin/bash
# Install Apache Web Server and PHP
yum install -y httpd mysql php

# Download Lab files
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RESTRT-1/267-lab-NF-build-vpc-web-server/s3/lab-app.zip
unzip lab-app.zip -d /var/www/html/

# Turn on web server
chkconfig httpd on
service httpd start

```

## 5. Best Practices & Security

* **Multi-AZ Architecture:** Always deploy critical workloads across at least two Availability Zones to ensure fault tolerance.
* **Public vs. Private:** Place resources that must be directly accessed from the internet (like Load Balancers or Bastion Hosts) in **Public Subnets**. Place backend servers, databases, and application servers in **Private Subnets**.
* **Secure Outbound Traffic:** Use NAT Gateways in your public subnets to allow your private instances to download updates securely without exposing them to inbound internet traffic.
* **Defense in Depth:** Combine Security Groups (which are stateful and act at the instance level) with Network ACLs (which are stateless and act at the subnet level) for granular security.

---