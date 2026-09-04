Aqui está a documentação completa para o laboratório de **AWS CloudTrail, Auditoria de Segurança, Análise de Logs via AWS Athena e Resposta a Incidentes**, baseada no roteiro fornecido.

Você pode salvar este arquivo em uma pasta de segurança ou auditoria do seu repositório, por exemplo, como `security/cloudtrail-incident-response.md`.

---

# AWS CloudTrail: Auditing, Log Analysis with Athena & Incident Response

## 1. Overview & Key Features

Security auditing and governance are critical capabilities in cloud architectures. **AWS CloudTrail** helps organizations track user activity and API usage across their AWS infrastructure. When security incidents occur—such as unauthorized security group modifications or compromised operating system users—CloudTrail logs provide the forensic trail required to identify the culprit, understand the attack vector, and remediate the vulnerability.

**Key Features:**

* **API Activity Tracking:** Record all management events, service calls, and console activities in an AWS account.
* **Secure Storage:** Automatically deliver immutable log files into an encrypted Amazon S3 bucket.
* **Interactive Querying (Athena):** Analyze massive volumes of CloudTrail JSON logs using standard SQL queries.
* **Forensic Investigation:** Trace source IP addresses, IAM user identities, timestamps, and request parameters associated with malicious actions.

## 2. Core Concepts

* **Trail:** A configuration that enables delivery of events as log files to an Amazon S3 bucket.
* **Management Events:** Provide visibility into management operations performed on resources in your AWS account (e.g., creating a security group, terminating an instance).
* **Athena Table Schema:** A predefined SQL table mapping JSON-formatted CloudTrail log structures into relational columns (`useridentity`, `eventname`, `eventsource`, `requestparameters`, etc.).
* **OS Hardening:** Securing Linux instances by disabling insecure authentication methods (such as root or password-based SSH logins) and enforcing key pair authentication.

---

## 3. Hands-On Guide: Investigating and Securing an Environment

### Task A: Create a CloudTrail Trail

1. Navigate to the **CloudTrail Console** > **Trails** > **Create trail**.
2. **Configuration:**
* **Trail name:** `monitor` *(Must match exactly for validation)*.
* **Storage location:** Select **Create a new S3 bucket** (e.g., `monitoring####`).
* **KMS encryption:** Specify or create a custom KMS alias (e.g., `kc-KMS`).


3. Complete the wizard and verify the trail is active. CloudTrail records management events and periodically delivers compressed JSON logs (`.json.gz`) to S3.

---

### Task B: Analyzing CloudTrail Logs via AWS Athena

Instead of manually parsing logs with Linux utilities (`grep`/`gunzip`), you can use Athena to query logs using standard SQL.

1. **Create the Athena Table:**
* In the CloudTrail console, navigate to **Event history** and click **Create Athena table**.
* Select your S3 logging bucket (`monitoring####`). Athena automatically generates the external table schema matching the JSON structure.


2. **Configure Athena Query Results:**
* Open the **Amazon Athena Console** > **Settings** > **Manage**.
* Set the Query Result Location to `s3://monitoring####/results/` and save.


3. **Run SQL Queries:**
* *Basic query to inspect log structure:*
```sql
SELECT useridentity.userName, eventtime, eventsource, eventname, requestparameters
FROM cloudtrail_logs_monitoring####
LIMIT 30;

```


* *Advanced query to find active users and actions over the past day (useful for isolating the hacker):*
```sql
SELECT DISTINCT useridentity.userName, eventName, eventSource 
FROM cloudtrail_logs_monitoring#### 
WHERE from_iso8601_timestamp(eventtime) > date_add('day', -1, now()) 
ORDER BY eventSource;

```





---

### Task C: Forensic Incident Response & Remediation

Once the investigation reveals the compromised IAM user (e.g., `chaos`) and the exact time port 22 was opened to the world (`0.0.0.0/0`), perform the following remediation steps:

#### 1. Evict Compromised OS Users

If a hacker created a local system user (e.g., `chaos-user`) on your EC2 instance:

* Identify active user processes:
```bash
who
sudo aureport --auth

```


* Kill the active login process using its Process ID (`ProcNum`):
```bash
sudo kill -9 <ProcNum>

```


* Delete the malicious OS user and their home directory:
```bash
sudo userdel -r chaos-user

```



#### 2. Harden SSH Security Configurations

Fix insecure SSH settings introduced during the breach:

1. Open the SSH configuration file:
```bash
sudo vi /etc/ssh/sshd_config

```


2. Ensure **Password Authentication** is disabled and key-based authentication is strictly enforced:
```text
PasswordAuthentication no

```


3. Restart the SSH daemon:
```bash
sudo service sshd restart

```



#### 3. Fix Cloud Infrastructure and IAM

1. **Remove the Network Security Hole:** Go to the EC2 console, locate your Web Server's Security Group, and delete the inbound rule allowing SSH (Port 22) from `0.0.0.0/0`. Restrict port 22 solely to your specific IP address (`/32`) or remove it entirely if using AWS Systems Manager Session Manager.
2. **Revoke Malicious IAM Access:** Go to the **IAM Console** > **Users**, select the rogue user (`chaos`), and **Delete** them to permanently strip programmatic and console access from the account.

---

## 4. Best Practices & Security

* **Enable CloudTrail Everywhere:** Always enable multi-region CloudTrail trails and apply log file integrity validation to ensure audit logs cannot be tampered with undetected.
* **Enforce Key-Based SSH:** Never allow `PasswordAuthentication yes` on production Linux servers. Rely strictly on cryptographic SSH key pairs or **AWS Systems Manager Session Manager** (which eliminates the need for open inbound ports or SSH keys altogether).
* **Least Privilege IAM:** Regularly review IAM user permissions and policies. Use automated tools like AWS IAM Access Analyzer to detect unintended public or cross-account access.

---