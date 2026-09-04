# Elastic Load Balancing (ELB) & EC2 Auto Scaling

## 1. Overview & Key Features

To achieve high availability and fault tolerance, modern cloud architectures must automatically adapt to changes in traffic.

* **Elastic Load Balancing (ELB)** automatically distributes incoming application traffic across multiple targets (like EC2 instances) in multiple Availability Zones.
* **Amazon EC2 Auto Scaling** helps you maintain application availability and allows you to automatically add or remove EC2 instances according to conditions you define.

**Key Features:**

* **Fault Tolerance:** If one instance fails health checks, the Load Balancer stops routing traffic to it, and Auto Scaling provisions a replacement.
* **Cost Optimization:** Scale out (add instances) during traffic spikes to maintain performance, and scale in (remove instances) during lulls to save money.
* **Multi-AZ Deployment:** Seamlessly span infrastructure across multiple Availability Zones to survive data center-level outages.

## 2. Core Concepts

* **AMI (Amazon Machine Image):** A snapshot of an EC2 instance's boot disk. Used as the baseline template to clone identical instances.
* **Application Load Balancer (ALB):** Operates at the request level (Layer 7), routing traffic to targets based on HTTP/HTTPS content.
* **Target Group:** A logical grouping of targets (like EC2 instances). The Load Balancer routes requests to registered targets in a Target Group.
* **Launch Template:** A configuration resource containing instance details (AMI, instance type, security groups, key pairs) used by Auto Scaling to launch new instances.
* **Auto Scaling Group (ASG):** A collection of EC2 instances treated as a logical grouping for the purposes of automatic scaling and management.
* **Target Tracking Scaling Policy:** A scaling policy that adds or removes instances to keep a specific metric (e.g., Average CPU Utilization) at a target value.

## 3. Hands-On Guide: Building a Scalable Architecture

### Task A: Create a Custom AMI (Golden Image)

Before you can scale, you need a template of your fully configured server.

1. In the EC2 Console, select an existing, fully configured instance (e.g., your Web Server).
2. Go to **Actions** > **Image and templates** > **Create image**.
3. Provide an **Image name** and **Create image**. This captures the OS, installed software, and configurations.

### Task B: Provision an Application Load Balancer

1. Navigate to **Load Balancers** and click **Create load balancer** (Choose Application Load Balancer).
2. **Network Mapping:** Select your VPC and at least two **Public Subnets** in different Availability Zones (so the ALB is highly available).
3. **Security Group:** Assign a Security Group that allows inbound HTTP/HTTPS traffic.
4. **Listeners and routing:** Create a new **Target Group** (Target type: Instances).
5. Associate the Target Group with the ALB listener and create the Load Balancer.

### Task C: Create a Launch Template

Auto Scaling needs instructions on *how* to build new instances.

1. Navigate to **Launch Templates** > **Create launch template**.
2. **AMI:** Select the custom AMI created in Task A.
3. **Instance type:** Choose the appropriate hardware (e.g., `t3.micro`).
4. **Network Settings:** Select the Security Group intended for the instances (e.g., Web Security Group). Do *not* define specific subnets here; the ASG will handle that.

### Task D: Configure the Auto Scaling Group (ASG)

1. Select your Launch Template and click **Create Auto Scaling group**.
2. **Network:** Choose your VPC and select at least two **Private Subnets** (instances should be private; the ALB acts as the public entry point).
3. **Load Balancing:** Choose **Attach to an existing load balancer** and select your Target Group. Enable **ELB Health Checks**.
4. **Group Size:** Set your capacities (e.g., Desired: 2, Minimum: 2, Maximum: 4).
5. **Scaling Policies:** Choose **Target tracking scaling policy**. Set the metric to **Average CPU utilization** with a target value of `50%`.

### Task E: Monitor and Stress Test

1. Navigate to **Target Groups** to verify the instances have successfully registered and their health status is **Healthy**.
2. Copy the Load Balancer's DNS name and paste it into a browser to access the application.
3. Navigate to **CloudWatch** > **Alarms**. You will see two alarms automatically created by the ASG (one for high CPU, one for low CPU).
4. If you intentionally stress the CPU of the instances, the `AlarmHigh` will trigger, and the ASG will launch additional instances to balance the load, up to your defined Maximum capacity.

## 4. Code Snippets & CLI Reference

**Optional Challenge: Create an AMI via AWS CLI**
If you want to automate the creation of your "Golden Image" rather than using the console, you can use the CLI:

```bash
# Command to create an AMI from a running EC2 instance
aws ec2 create-image \
    --instance-id i-1234567890abcdef0 \
    --name "Web-Server-AMI-CLI" \
    --description "AMI created via AWS CLI for Auto Scaling"

```

*Note: This command returns an AMI ID (e.g., `ami-0abcdef1234567890`), which you can then pass programmatically into a Launch Template creation script.*

## 5. Best Practices & Security

* **Private Compute, Public Routing:** Always place your Auto Scaling EC2 instances in **Private Subnets**. Only the Application Load Balancer should reside in the **Public Subnets**.
* **ELB Health Checks:** When configuring an ASG attached to a Load Balancer, always switch the ASG health check type from `EC2` to `ELB`. This ensures that if an instance's web server crashes (but the OS is still running), the ASG will terminate and replace it.
* **Stateless Applications:** Auto Scaling works best when instances are stateless. Store session data in a database (like DynamoDB) or an in-memory cache (like ElastiCache) rather than on the local disk, as instances can be terminated at any time.

---