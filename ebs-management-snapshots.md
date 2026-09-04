Aqui está a documentação para **Gerenciamento de Armazenamento (EBS, Snapshots Automatizados via Cron/Python e S3 Sync com Versionamento)**, baseada no laboratório fornecido.

Você pode salvar este arquivo na pasta de armazenamento do repositório, por exemplo, como `storage/storage-management-snapshots-s3.md`.

---

# Advanced Storage Management: EBS Snapshots & S3 Synchronization

## 1. Overview & Key Features

Managing data persistence in the cloud requires robust backup strategies and efficient synchronization between block storage (Amazon EBS) and object storage (Amazon S3).

This guide covers automating EBS volume snapshots using Linux cron jobs and custom Python scripts for retention control, as well as synchronizing local EBS directories to an S3 bucket with **S3 Versioning** enabled for disaster recovery.

**Key Features:**

* **Automated Backups:** Use cron scheduling to create point-in-time EBS snapshots automatically.
* **Lifecycle Management (Retention):** Implement scripts to prune older snapshots and retain only the most critical recovery points.
* **S3 Synchronization:** Efficiently replicate local files to Amazon S3 using the `aws s3 sync` utility.
* **Version Control & Data Recovery:** Use S3 Versioning and delete markers to recover accidentally removed files.

## 2. Core Concepts

* **Instance Profile:** An IAM role attached directly to an EC2 instance, enabling the instance's OS/scripts to securely invoke AWS APIs (like EC2 and S3) without storing hardcoded credentials.
* **Cron / Crontab:** A time-based job scheduler in Unix-like operating systems used to run automated scripts or commands at fixed intervals.
* **S3 Versioning:** A means of keeping multiple variants of an object in the same bucket, protecting against accidental overwrites or deletions by inserting a "delete marker".
* **`aws s3 sync`:** A command-line utility that synchronizes local directories with S3 buckets, supporting mirroring flags like `--delete`.

## 3. Hands-On Guide: Storage Automation & S3 Sync

### Task A: Configure Permissions (Instance Profile)

Before scripts can interact with AWS services, the target EC2 instance (`Processor`) must have the correct permissions.

1. Navigate to the **EC2 Console** > **Instances**.
2. Select the `Processor` instance.
3. Go to **Actions** > **Security** > **Modify IAM role**.
4. Select the pre-created role (e.g., `S3BucketAccess`) and update the IAM role.

### Task B: Automating and Managing EBS Snapshots via CLI

Using the `Command Host` instance, you can programmatically manage block storage backups.

1. **Identify the EBS Volume ID:**
```bash
aws ec2 describe-instances --filter 'Name=tag:Name,Values=Processor' --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.{VolumeId:VolumeId}'

```


2. **Stop the Instance (Best Practice for Consistent File System Backups):**
```bash
INSTANCE_ID=$(aws ec2 describe-instances --filters 'Name=tag:Name,Values=Processor' --query 'Reservations[0].Instances[0].InstanceId' --output text)
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-stopped --instance-id $INSTANCE_ID

```


3. **Take an Initial Snapshot:**
```bash
aws ec2 create-snapshot --volume-id <VOLUME-ID>

```


*Restart the instance afterwards (`aws ec2 start-instances --instance-ids $INSTANCE_ID`).*
4. **Automate via Cron (Simulating Frequent Backups):**
To schedule a snapshot every minute using a cron job:
```bash
echo "* * * * *  aws ec2 create-snapshot --volume-id <VOLUME-ID> 2>&1 >> /tmp/cronlog" > cronjob
crontab cronjob

```


*To stop the cron job when finished:* `crontab -r`
5. **Retention Management (Pruning Old Snapshots):**
Running cron jobs rapidly accumulates snapshots. Use a Python maintenance script (`snapshotter_v2.py`) to sort snapshots by date and automatically prune everything except the two most recent versions:
```bash
python3.8 snapshotter_v2.py

```



### Task C: Synchronizing EBS Files to Amazon S3 with Versioning

To protect data across storage tiers, mirror an EBS directory to an S3 bucket.

1. **Create an S3 Bucket:**
```bash
aws s3api create-bucket --bucket <s3-bucket-name> --region 'us-west-2' \
    --create-bucket-configuration LocationConstraint='us-west-2'

```


2. **Enable S3 Versioning:**
```bash
aws s3api put-bucket-versioning --bucket <s3-bucket-name> --versioning-configuration Status=Enabled

```


3. **Download and Unzip Sample Data (on the Processor instance):**
```bash
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-3-124627/183-lab-JAWS-managing-storage/s3/files.zip
unzip files.zip

```


4. **Sync Local Files to S3:**
```bash
aws s3 sync files s3://<s3-bucket-name>/files/

```


5. **Mirror Deletions (`--delete`):**
If a file is deleted locally (e.g., `rm files/file1.txt`), you can force S3 to reflect the deletion by adding the `--delete` flag to the sync command:
```bash
aws s3 sync files s3://<s3-bucket-name>/files/ --delete

```



### Task D: Recovering Deleted Files via S3 Versioning

When versioning is enabled, deleting a file in S3 doesn't destroy it permanently; instead, S3 adds a **Delete Marker**.

1. **List Object Versions to find the target file's ID:**
```bash
aws s3api list-object-versions --bucket <s3-bucket-name> --prefix files/file1.txt

```


*Locate the specific non-delete version ID of `file1.txt`.*
2. **Download the Older Version:**
```bash
aws s3api get-object --bucket <s3-bucket-name> --key files/file1.txt --version-id <VERSION-ID> files/file1.txt

```


3. **Re-sync to Restore:**
```bash
aws s3 sync files s3://<s3-bucket-name>/files/

```



## 4. Best Practices & Security

* **Consistency vs. Availability:** For transactional databases or busy file systems, always stop the EC2 instance or flush memory buffers (`fsfreeze`) before taking an EBS snapshot to ensure application-consistent backups.
* **Snapshot Retention Policies:** Never let cron jobs create unlimited snapshots indefinitely. Implement automated scripts or leverage **AWS Backup / Amazon DLM (Data Lifecycle Manager)** to enforce retention limits and control S3 storage costs.
* **S3 Versioning Safeguards:** Enable versioning on production S3 buckets to protect against accidental deletions and ransomware attacks. Combine versioning with **Lifecycle Rules** to transition older versions to Glacier or delete expired delete markers over time.

---