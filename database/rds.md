Here is the documentation for **Amazon Relational Database Service (RDS)** 
---

# Amazon Relational Database Service (RDS)

## 1. Overview & Key Features

Amazon Relational Database Service (Amazon RDS) is a managed web service that makes it easy to set up, operate, and scale a relational database in the cloud. It provides cost-efficient and resizable capacity while automating time-consuming administrative tasks such as hardware provisioning, database setup, patching, and backups.

**Key Features:**

* **Multiple Engines:** Supports Amazon Aurora, PostgreSQL, MySQL, MariaDB, Oracle, and Microsoft SQL Server.
* **High Availability:** Multi-AZ deployments automatically provision and maintain a synchronous standby replica in a different Availability Zone.
* **Automated Management:** Handles automated backups, software patching, and automatic failure detection and recovery.
* **Security:** Allows you to isolate your database in a VPC and control access using IAM and Security Groups.

## 2. Core Concepts

* **DB Instance:** An isolated database environment running in the cloud, representing your database server.
* **Endpoint:** The DNS address provided by AWS that applications use to connect to your database instance.
* **DB Subnet Group:** A collection of subnets (typically private) spanning multiple Availability Zones that you create in a VPC and designate for your RDS DB instances.
* **Multi-AZ Deployment:** A deployment method where RDS automatically creates a primary DB instance and synchronously replicates the data to a standby instance in a different AZ for failover.

## 3. Hands-On Guide: Deploying a Multi-AZ Database

### Task A: Configure the Database Security Group

To secure the database, you must configure a Security Group that acts as a firewall, allowing access only from the application tier.

1. Navigate to the **VPC Dashboard** and select **Security groups**.
2. Create a new Security Group (e.g., `DB Security Group`) attached to your custom VPC.
3. Add an **Inbound Rule**:
* Type: `MySQL/Aurora (3306)`
* Source: Specify the ID of the **Web Security Group** used by your EC2 instances. This ensures only your web servers can communicate with the database.



### Task B: Create a DB Subnet Group

RDS requires a DB Subnet Group to know which subnets it can use to deploy the database across multiple Availability Zones.

1. Navigate to the **RDS Dashboard** and choose **Subnet groups**.
2. Create a new DB Subnet Group.
3. Select your custom VPC.
4. Add at least two **Private Subnets** located in different Availability Zones to support High Availability.

### Task C: Launch the RDS DB Instance

1. Go to the **RDS Dashboard** > **Databases** > **Create database**.
2. Select **Standard create** and choose your engine (e.g., `MySQL`).
3. Under Availability and durability, select **Multi-AZ DB instance deployment** for production-grade reliability.
4. Define your **Master username** and **Master password**.
5. Select an instance class (e.g., `db.t3.medium`) and allocate storage (e.g., `20 GB gp3`).
6. Under Connectivity, select your custom VPC, the DB Subnet Group, and the `DB Security Group` created in Task A. Set **Public access** to `No`.
7. Expand **Additional configuration** to specify an Initial database name (e.g., `lab`).
8. Launch the database and wait for its status to become **Available**. Copy the generated **Endpoint**.

### Task D: Interact with the Database via an Application

Once the database is available, connect your application using the provided credentials.

1. Access your web server's application interface via your browser.
2. Provide the database connection details:
* **Endpoint:** (e.g., `lab-db.xxxxx.us-west-2.rds.amazonaws.com`)
* **Database:** The initial DB name (`lab`)
* **Username & Password:** Your master credentials.


3. Test read/write operations through the app to verify successful connectivity.

## 4. Configuration Quick Reference

**Standard Database Connection Parameters:**
When configuring applications (like WordPress or custom scripts) to connect to RDS, you will typically need these four variables mapped to your environment:

* `DB_HOST`: The RDS Endpoint (Do not include `http://` or port numbers here unless requested).
* `DB_NAME`: The initial database name created during setup.
* `DB_USER`: The master username.
* `DB_PASSWORD`: The master password.

## 5. Best Practices & Security

* **Keep Databases Private:** Always deploy RDS instances in **Private Subnets** with **Public access** set to `No`.
* **Security Group Referencing:** Instead of allowing specific IP addresses, configure your Database Security Group to allow traffic strictly from your Application's Security Group ID.
* **Enable Multi-AZ:** Use Multi-AZ deployments for any production workload to ensure automatic failover in case of hardware or Availability Zone degradation.
* **Backups:** Always leave **Enable automated backups** turned on for production (it may be disabled in lab environments to speed up deployment).

---