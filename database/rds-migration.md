Esta documentação aborda o processo completo de migração de uma arquitetura baseada em instância única (LAMP) para um banco de dados totalmente gerenciado, incluindo criação de sub-redes privadas, grupos de segurança com least-privilege, dump/restore de dados via CLI e configuração de parâmetros via Systems Manager.
---

# Database Migration to Amazon RDS (MariaDB)

## 1. Overview & Key Features

Migrating from a self-managed database running on an Amazon EC2 instance (such as a traditional LAMP stack) to a fully managed service like **Amazon Relational Database Service (RDS)** reduces administrative overhead, enhances reliability, and simplifies scaling.

**Key Features:**

* **Fully Managed Infrastructure:** Automated patching, backups, hardware provisioning, and failure recovery.
* **High Availability & Security:** Easily deploy multi-AZ instances, enforce SSL/TLS encrypted connections, and isolate databases inside private subnets.
* **Performance Monitoring:** Native integration with Amazon CloudWatch for real-time metrics (CPU, IOPS, Connections, Memory).
* **Externalized Configurations:** Decoupling application connection strings using Parameter Store.

## 2. Core Concepts

* **DB Subnet Group:** A collection of subnets (typically private) that you designate for your RDS DB instances across multiple Availability Zones in a VPC.
* **Database Security Group:** Acts as a virtual firewall for the RDS instance. In a best-practice architecture, it only allows inbound traffic on port 3306 from the specific Security Group assigned to the Application/EC2 instances (preventing public exposure).
* **`mysqldump` & `mysql`:** Standard command-line tools used to extract (dump) SQL schemas and data from a source database and import (restore) them into a target database.
* **SSL/TLS Certificate Bundle (`global-bundle.pem`):** Required by modern managed databases to validate and encrypt client connections securely.

## 3. Hands-On Guide: Migrating from EC2 to RDS

### Task A: Provision Prerequisite Infrastructure via AWS CLI

Before creating the RDS instance, you must configure the networking and security boundaries.

1. **Create the Database Security Group (`CafeDatabaseSG`):**
Ensure only instances attached to the web server's security group can access the database port (`3306`).
```bash
aws ec2 create-security-group \
    --group-name CafeDatabaseSG \
    --description "Security group for Cafe database" \
    --vpc-id <CafeVpcID>

```


2. **Authorize Inbound Traffic:**
Allow MySQL traffic strictly from the web server's Security Group ID:
```bash
aws ec2 authorize-security-group-ingress \
    --group-id <CafeDatabaseSG-GroupId> \
    --protocol tcp --port 3306 \
    --source-group <CafeSecurityGroup-GroupId>

```


3. **Create Private Subnets:**
Create two empty private subnets in different Availability Zones within the VPC CIDR block (e.g., `10.200.2.0/23` and `10.200.10.0/23`).
```bash
aws ec2 create-subnet --vpc-id <CafeVpcID> --cidr-block 10.200.2.0/23 --availability-zone us-west-2a
aws ec2 create-subnet --vpc-id <CafeVpcID> --cidr-block 10.200.10.0/23 --availability-zone us-west-2b

```


4. **Create the DB Subnet Group:**
Register the private subnets into a DB Subnet Group for RDS:
```bash
aws rds create-db-subnet-group \
    --db-subnet-group-name "CafeDB Subnet Group" \
    --db-subnet-group-description "DB subnet group for Cafe" \
    --subnet-ids <SubnetId-1> <SubnetId-2> \
    --tags "Key=Name,Value=CafeDatabaseSubnetGroup"

```



### Task B: Launch the Amazon RDS MariaDB Instance

Use the CLI to deploy the managed database instance into the private subnets.

```bash
aws rds create-db-instance \
    --db-instance-identifier CafeDBInstance \
    --engine mariadb \
    --engine-version 10.11.11 \
    --db-instance-class db.t3.micro \
    --allocated-storage 20 \
    --availability-zone us-west-2a \
    --db-subnet-group-name "CafeDB Subnet Group" \
    --vpc-security-group-ids <CafeDatabaseSG-GroupId> \
    --no-publicly-accessible \
    --master-username root \
    --master-user-password 'Re:Start!9'

```

*Wait until the instance status transitions from `creating` to `available`. Copy its generated **Endpoint Address**.*

### Task C: Backup and Migrate Data (`mysqldump` & `mysql`)

1. **Connect to the Application Instance** (via EC2 Instance Connect).
2. **Export the Local Database:**
Generate a SQL dump file of the local `cafe_db` database.
```bash
mysqldump --user=root --password='Re:Start!9' --databases cafe_db --add-drop-database > cafedb-backup.sql

```


3. **Download the RDS SSL Certificate Bundle:**
```bash
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

```


4. **Import Data into the RDS Instance:**
Restore the dump file directly to the remote RDS endpoint using encrypted connections.
```bash
mysql --user=root --password='Re:Start!9' \
    --host=<RDS-Endpoint-Address> \
    --ssl-ca=./global-bundle.pem \
    < cafedb-backup.sql

```



### Task D: Reconfigure the Application (Parameter Store)

Instead of modifying application code files, update the centralized configuration.

1. Navigate to **AWS Systems Manager** > **Parameter Store**.
2. Select the parameter `/cafe/dbUrl`.
3. Choose **Edit** and replace the old local database IP/URL with the **RDS Database Endpoint Address**.
4. Save changes. The application will instantly begin interacting with the managed database.

### Task E: Monitor Performance via CloudWatch

1. Open the **Amazon RDS Console** > **Databases** > `cafedbinstance`.
2. Choose the **Monitoring** tab to view real-time performance graphs:
* `DatabaseConnections`: Track active sessions.
* `CPUUtilization`: Monitor compute load.
* `ReadIOPS` / `WriteIOPS`: Measure disk throughput.



## 4. Best Practices & Security

* **Never Make Databases Public:** Always set `--no-publicly-accessible` when creating production databases. Access should flow through load balancers, application servers, or bastion hosts.
* **Security Group Referencing:** Restrict database access using Security Group IDs rather than hardcoded IP address ranges (`0.0.0.0/0`), ensuring tight boundaries between tiers.
* **Enforce SSL/TLS:** Always configure applications to use certificates (`--ssl-ca`) when communicating with managed databases to protect data in transit.
* **Parameterize Connections:** Externalize database credentials and endpoints using **AWS Systems Manager Parameter Store** or **Secrets Manager** to maintain clean code and agile environment transitions.

---