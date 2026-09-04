Documentação completa para a **Configuração de VPC (Virtual Private Cloud)**, cobrindo o design de redes isoladas, sub-redes públicas e privadas, gateways de internet, NAT Gateways e o uso de Bastion Hosts para acesso seguro.
---

# Amazon VPC: Custom Architecture, Subnets & NAT Gateways

## 1. Overview & Key Features

Amazon Virtual Private Cloud (Amazon VPC) lets you provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define.

To build a secure architecture, workloads are distributed across **Public Subnets** (accessible via the internet) and **Private Subnets** (isolated from direct internet inbound access).

**Key Features:**

* **Complete Network Control:** Define CIDR blocks, custom route tables, and security boundaries.
* **Internet Gateway (IGW):** Enables inbound and outbound traffic between resources in public subnets and the internet.
* **NAT Gateway:** Allows resources in private subnets to initiate outbound connections to the internet (for software updates or patches) while blocking unrequested inbound connections.
* **Bastion Host (Jump Box):** A securely hardened instance placed in a public subnet used by operators to tunnel into private instances.

## 2. Core Concepts

* **CIDR Block (Classless Inter-Domain Routing):** The IP address range assigned to your VPC (e.g., `10.0.0.0/16`) and its subnets (e.g., Public: `10.0.0.0/24`, Private: `10.0.2.0/23`).
* **Public Subnet:** Associated with a route table that directs internet-bound traffic (`0.0.0.0/0`) to an Internet Gateway.
* **Private Subnet:** Associated with a route table that directs internet-bound traffic to a NAT Gateway instead of an IGW.
* **Elastic IP (EIP):** A static public IPv4 address required when provisioning a NAT Gateway so its outbound IP address remains constant.

## 3. Hands-On Guide: Building a Custom VPC

### Task A: Create the VPC

1. Navigate to the **VPC Dashboard** > **Your VPCs** > **Create VPC**.
2. **Settings:**
* **Resources to create:** `VPC only`
* **Name tag:** `Lab VPC`
* **IPv4 CIDR:** `10.0.0.0/16`


3. Click **Create VPC**.
4. *Crucial Step:* Select your new VPC, go to **Actions** > **Edit VPC settings**, and ensure **Enable DNS hostnames** is checked so instances receive public DNS names automatically.

### Task B: Create Public and Private Subnets

1. **Public Subnet:**
* Go to **Subnets** > **Create subnet**. Select `Lab VPC`.
* Name: `Public Subnet`, AZ: First Availability Zone, CIDR: `10.0.0.0/24`.
* *Auto-assign Public IP:* Select the subnet, go to **Actions** > **Edit subnet settings**, and enable **Auto-assign public IPv4 address**.


2. **Private Subnet:**
* Create another subnet in the same VPC and AZ.
* Name: `Private Subnet`, CIDR: `10.0.2.0/23` (a larger block allocated for backend resources).



### Task C: Attach an Internet Gateway (IGW)

1. Go to **Internet gateways** > **Create internet gateway** (Name: `Lab IGW`).
2. Select the created IGW, go to **Actions** > **Attach to a VPC**, and choose `Lab VPC`.

### Task D: Configure Route Tables

1. **Private Route Table:**
* Locate the default route table associated with your VPC and rename it to `Private Route Table`. (By default, it handles local VPC communication `10.0.0.0/16`).


2. **Public Route Table:**
* Go to **Route tables** > **Create route table** (Name: `Public Route Table`, VPC: `Lab VPC`).
* Edit routes, add a route with Destination `0.0.0.0/0` targeting your `Lab IGW`.
* Go to the **Subnet associations** tab, edit associations, and select `Public Subnet`.



### Task E: Deploy a Bastion Server in the Public Subnet

1. Go to the **EC2 Dashboard** > **Launch instances**.
2. **Name:** `Bastion Server` (AMI: Amazon Linux 2023, Type: `t3.micro`).
3. **Network Settings:** Select `Lab VPC`, choose `Public Subnet`, and enable **Auto-assign public IP**.
4. **Security Group:** Create a security group named `Bastion Security Group` with an inbound rule allowing **SSH (Port 22)** from **Anywhere**.
5. Launch the instance.

### Task F: Deploy a NAT Gateway for Private Subnet Outbound Access

1. Search for **NAT gateways** in the AWS Console and click **Create NAT gateway**.
2. **Settings:** Name: `Lab NAT gateway`, Subnet: `Public Subnet`. Click **Allocate Elastic IP** and create.
3. **Route Traffic:** Go back to **Route tables**, select `Private Route Table`, edit routes, and add a route for Destination `0.0.0.0/0` pointing to your new **NAT Gateway**.

## 4. Testing Private Connectivity (Optional Challenge)

To verify that your private subnet and NAT gateway work properly:

1. **Launch a Private Instance:** Create an EC2 instance in the `Private Subnet` with a security group allowing SSH traffic exclusively from the VPC CIDR range (`10.0.0.0/16`).
2. **Jump via Bastion:**
* Connect to your **Bastion Server** using *EC2 Instance Connect*.
* From inside the Bastion terminal, use SSH to log into the private instance using its private IP address:


```bash
ssh 10.0.2.xxx

```


3. **Test Internet Access:** From the private instance, ping an external domain:
```bash
ping -c 3 amazon.com

```


*If responses are returned, it confirms the private instance successfully routed its traffic outward through the NAT Gateway.*

## 5. Best Practices & Security

* **Principle of Least Privilege for SSH:** Never leave SSH (Port 22) open to `0.0.0.0/0` on production instances. Restrict access to specific corporate IP addresses or strictly route traffic through a Bastion Host / AWS Systems Manager Session Manager.
* **Subnet Sizing:** Allocate larger CIDR blocks to private subnets (e.g., `/23` or `/22`) where your application servers, databases, and microservices live, while keeping public subnets (`/24`) restricted strictly to Load Balancers and Bastion hosts.
* **Multi-AZ Redundancy:** In production environments, deploy your public and private subnets across at least **two Availability Zones** to ensure high availability and fault tolerance.

---