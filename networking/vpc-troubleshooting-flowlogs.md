Esta documentação aborda metodologias de diagnóstico de rede no nível da AWS (rotas, ACLs, Security Groups) e técnicas de auditoria utilizando **VPC Flow Logs** armazenados no Amazon S3 e analisados via CLI (`grep`/`gunzip`).
---

# Amazon VPC: Advanced Troubleshooting & VPC Flow Logs Analysis

## 1. Overview & Key Features

When deploying multi-tier applications in an Amazon VPC, configuration errors (such as missing routes, misconfigured network ACLs, or overly restrictive security groups) can block access to resources.

**VPC Flow Logs** capture information about the IP traffic going to and from network interfaces in your VPC, providing a powerful audit trail for troubleshooting connectivity issues and analyzing security incidents.

**Key Features:**

* **Deep Visibility:** Capture IP traffic data at the ENI (Elastic Network Interface) level.
* **Security Auditing:** Track allowed and rejected traffic (`ACCEPT` / `REJECT`) to identify unauthorized access attempts.
* **Flexible Destination:** Publish flow logs directly to **Amazon S3** or **Amazon CloudWatch Logs** for long-term storage and analysis.
* **Programmatic Troubleshooting:** Diagnose network paths rapidly using the AWS CLI combined with Linux text-processing utilities (`grep`, `awk`, `gunzip`).

## 2. Core Concepts

* **VPC Flow Log:** A feature that enables you to capture information about IP traffic flowing to and from network interfaces in your VPC.
* **Traffic Types:** You can log `ACCEPT` traffic, `REJECT` traffic, or `ALL` traffic.
* **Network ACL (NACL):** A stateless, subnet-level virtual firewall that evaluates inbound and outbound rules.
* **Security Group:** A stateful, instance-level virtual firewall.
* **Route Table:** A set of rules (routes) used to determine where network traffic is directed from subnets.

## 3. Hands-On Guide: Troubleshooting & Flow Log Analysis

### Task A: Create and Configure VPC Flow Logs

To capture network behavior, set up a flow log pointing to an Amazon S3 bucket.

1. **Create an S3 Bucket:**
```bash
aws s3api create-bucket --bucket flowlog###### --region 'us-west-2' \
    --create-bucket-configuration LocationConstraint='us-west-2'

```


2. **Retrieve the Target VPC ID:**
```bash
aws ec2 describe-vpcs --filters "Name=tag:Name,Values='VPC1'" --query 'Vpcs[*].VpcId' --output text

```


3. **Create the Flow Log:**
Capture all traffic (`ALL`) directed to the VPC and send it to the S3 bucket:
```bash
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-id <vpc-id> \
    --traffic-type ALL \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::<flowlog######>

```



---

### Task B: Troubleshooting Challenge #1 - Missing Internet Route

**Symptom:** An EC2 instance is running inside a public subnet, but its web server page (`http://<WebServerIP>`) fails to load (connection times out).

**Diagnosis & Resolution via CLI:**

1. **Check Open Ports:** Install `nmap` on a diagnostic host and scan the web server:
```bash
sudo yum install -y nmap
nmap -Pn <WebServerIP>

```


*If ports appear filtered or closed even though the instance is running, the issue is likely at the network route or firewall level.*
2. **Inspect Route Tables:** Verify if the subnet has a valid route to the Internet Gateway (`0.0.0.0/0`).
```bash
aws ec2 describe-route-tables --route-table-ids '<VPC1PubRouteTableId>' \
    --filter "Name=association.subnet-id,Values='<VPC1PubSubnetID>'"

```


3. **Fix the Route:** If the default route (`0.0.0.0/0`) pointing to the Internet Gateway is missing, add it:
```bash
aws ec2 create-route \
    --route-table-id '<VPC1PubRouteTableId>' \
    --gateway-id '<VPC1GatewayId>' \
    --destination-cidr-block '0.0.0.0/0'

```



---

### Task C: Troubleshooting Challenge #2 - Network ACL Blocking SSH

**Symptom:** Web traffic works, but attempting to connect via SSH (Port 22) or EC2 Instance Connect fails with a timeout error.

**Diagnosis & Resolution via CLI:**

1. **Inspect Network ACLs:** Check the inbound/outbound rules associated with the subnet. NACLs are stateless and process rules in numerical order.
```bash
aws ec2 describe-network-acls \
    --filter "Name=association.subnet-id,Values='<VPC1PublicSubnetID>' \
    --query 'NetworkAcls[*].[NetworkAclId,Entries]'

```


2. **Remove Blocking Rules:** If an explicit `DENY` rule (e.g., Rule Number 40) blocks traffic, delete that specific rule:
```bash
aws ec2 delete-network-acl-entry \
    --network-acl-id '<acl-id>' \
    --ingress \
    --rule-number 40

```



---

### Task D: Analyzing VPC Flow Logs in S3

Once the issues are resolved, you can analyze historical connection rejections (such as your failed SSH attempts) stored in S3.

1. **Download Flow Logs from S3:**
```bash
mkdir flowlogs && cd flowlogs
aws s3 cp s3://<flowlog######>/ . --recursive

```


2. **Extract Compressed Logs:**
Navigate down the nested S3 folder structure (`AWSLogs/<AccountID>/vpcflowlogs/us-west-2/yyyy/mm/dd/`) and unzip the files:
```bash
gunzip *.gz

```


3. **Query Log Entries (`grep`):**
* *Search for all rejected traffic:*
```bash
grep -rn REJECT .

```


* *Isolate rejected attempts on Port 22 (SSH) originating from your specific local IP address:*
```bash
grep -rn 22 . | grep REJECT | grep <your-local-ip>

```




4. **Translate Unix Timestamps:**
Flow log timestamps appear in Unix format (e.g., `1554496931`). Convert them to a human-readable date to pinpoint when the failed attempts occurred:
```bash
date -d @1554496931

```



## 4. Best Practices & Security

* **Stateless vs. Stateful:** Remember that Security Groups are *stateful* (return traffic is automatically allowed), while Network ACLs are *stateless* (you must explicitly allow both inbound and outbound traffic in NACL rules).
* **Flow Log Destinations:** While S3 is ideal for long-term auditing and data lake integration (using tools like Amazon Athena), publishing flow logs to **Amazon CloudWatch Logs** enables real-time monitoring and metric filtering for security alerts.
* **Log Retention:** Configure S3 Lifecycle Policies on your flow log buckets to automatically transition older logs to Glacier or delete them after a compliance period to manage storage costs.

---