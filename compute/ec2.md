# Amazon Elastic Compute Cloud (EC2)

## 1. Overview & Key Features

Amazon EC2 provides scalable computing capacity in the Amazon Web Services (AWS) Cloud. It allows you to develop and deploy applications faster without needing to invest in hardware up front. You can use Amazon EC2 to launch as many or as few virtual servers as you need, configure security and networking, and manage storage.

**Key Features:**

* **Virtual Servers:** Known as instances.
* **Pre-configured Templates:** Amazon Machine Images (AMIs) package the bits you need for your server (including the operating system and additional software).
* **Various Configurations:** Instance types offer varying combinations of CPU, memory, storage, and networking capacity.
* **Secure Computing:** Security groups act as virtual firewalls to control inbound and outbound traffic.

## 2. Core Concepts

* **AMI (Amazon Machine Image):** A template containing the software configuration (OS, application server, and applications) required to launch an instance.
* **Instance Type:** Determines the hardware of the host computer used for your instance (e.g., `t3.micro` for general purpose, 2 vCPUs, 1 GiB memory).
* **User Data:** Scripts or commands you can run on your instance at launch time to automate configuration tasks.
* **EBS (Elastic Block Store):** Persistent block-level storage volumes for use with EC2 instances (acts like a virtual hard drive).
* **Security Group:** A virtual firewall that controls the traffic for one or more instances.
* **Termination Protection:** A safety feature that prevents instances from being accidentally terminated.

## 3. Hands-On Guide: Managing an EC2 Instance

### Task A: Launching an Instance with a Web Server

1. Navigate to the **EC2 Dashboard** and click **Launch Instance**.
2. **Name:** Assign a tag (e.g., `Web Server`).
3. **AMI:** Select an OS (e.g., *Amazon Linux 2023*).
4. **Instance Type:** Choose your hardware (e.g., `t3.micro`).
5. **Key Pair:** Used for SSH access. (Can proceed without one if SSH is not needed).
6. **Network & Security:** Select your VPC. Create or assign a **Security Group**.
7. **Storage:** Leave the default EBS volume (e.g., 8 GiB) or adjust as needed.
8. **Advanced Details:**
* Enable **Termination protection**.
* Add a **User Data** script to bootstrap the server.



### Task B: Monitoring the Instance

* **Status Checks:** EC2 automatically performs checks.
* *System reachability:* Checks the underlying AWS hardware.
* *Instance reachability:* Checks the software/network configuration of your specific instance.


* **CloudWatch Metrics:** The **Monitoring** tab provides basic (5-minute) or detailed (1-minute) metrics.
* **Instance Screenshot:** A useful troubleshooting tool (Actions > Monitor and troubleshoot > Get Instance Screenshot) if you lose SSH/RDP access.

### Task C: Updating Security Groups (Firewall)

By default, a new web server won't be accessible from the internet unless you explicitly allow it.

1. Go to **Security Groups** in the left navigation pane.
2. Select your instance's security group.
3. Edit **Inbound Rules**.
4. Add a rule: Type `HTTP`, Source `Anywhere-IPv4` (Port 80).
5. Access the Public IPv4 address in a browser to see your running application.

### Task D: Resizing Instances and EBS Volumes

You can scale your resources vertically if they are over-utilized or under-utilized.

* **Change Instance Type (CPU/RAM):**
1. **Stop** the instance (Requires a stopped state).
2. Go to Actions > Instance Settings > **Change Instance Type** (e.g., to `t3.small`).
3. **Start** the instance.


* **Resize EBS Volume (Storage):**
1. Navigate to **Volumes** under Elastic Block Store.
2. Select the attached volume > **Modify Volume**.
3. Increase the size (e.g., from 8 GiB to 10 GiB) and confirm.



### Task E: Termination and Protection

* If **Termination Protection** is enabled, attempts to terminate the instance will fail.
* To delete the instance:
1. Go to Actions > Instance settings > **Change termination protection**.
2. Uncheck **Enable** and Save.
3. Go to Instance State > **Terminate instance**. *(Note: By default, the root EBS volume is deleted upon termination).*



## 4. Code Snippets

**User Data Script (Apache Web Server Setup):**
This bash script installs an Apache web server, configures it to start on boot, and creates a basic HTML page.

```bash
#!/bin/bash
yum -y install httpd
systemctl enable httpd
systemctl start httpd
echo '<html><h1>Hello From Your Web Server!</h1></html>' > /var/www/html/index.html

```

## 5. Best Practices & Security

* **Least Privilege:** Only open ports in your Security Groups that are absolutely necessary (e.g., open Port 80 for HTTP, but restrict Port 22/SSH to your specific IP address).
* **Prevent Accidents:** Always enable **Termination Protection** for production instances.
* **Automate Initialization:** Use **User Data** to deploy software automatically so your servers are identical and easily replaceable.
* **Cost Management:** Stop instances when not in use (you are not charged for compute time on a stopped instance, only for the attached EBS storage).

---