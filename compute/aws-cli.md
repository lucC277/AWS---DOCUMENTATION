# AWS Command Line Interface (CLI)

## 1. Overview & Key Features

The AWS Command Line Interface (AWS CLI) is a unified tool to manage your AWS services. With just one tool to download and configure, you can control multiple AWS services from the command line and automate them through scripts.

While some Amazon Machine Images (like Amazon Linux) come with the AWS CLI pre-installed, others (like Red Hat or Ubuntu) require manual installation.

**Key Features:**

* **Automation:** Script your infrastructure deployments and management tasks.
* **Direct Service Access:** Interact with almost all AWS services (EC2, S3, IAM, VPC, etc.) directly from your terminal.
* **Cross-Platform:** Available for Linux, macOS, and Windows.
* **Formatted Output:** Supports multiple output formats such as JSON, text, and table for easier parsing.

## 2. Core Concepts

* **SSH (Secure Shell):** A cryptographic network protocol used to securely connect to a remote server (e.g., an EC2 instance) over an unsecured network.
* **Access Key ID & Secret Access Key:** Long-term credentials for an IAM user, used to authenticate programmatic requests to AWS (like those made by the CLI).
* **Default Region:** The geographic AWS Region (e.g., `us-west-2`) where your CLI commands will be sent if no region is explicitly specified in the command.
* **Output Format:** The format in which the CLI returns information (JSON is the default and most commonly used).

## 3. Hands-On Guide: Installing and Configuring AWS CLI

### Task A: Connect to a Linux EC2 Instance via SSH

To install the CLI on a remote server, you must first connect to it.

* **Mac/Linux:** Use the native terminal.
```bash
chmod 400 labsuser.pem
ssh -i labsuser.pem ec2-user@<public-ip-address>

```


* **Windows:** Use a tool like **PuTTY**. Convert the `.pem` key to `.ppk`, input the instance's Public IP, and open the SSH connection.

### Task B: Install the AWS CLI (Red Hat / CentOS / Amazon Linux)

Once connected to the instance, download and install the AWS CLI version 2.

1. Download the installation file:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

```


2. Unzip the downloaded file:
```bash
unzip -u awscliv2.zip

```


3. Run the installer:
```bash
sudo ./aws/install

```


4. Verify the installation:
```bash
aws --version

```



### Task C: Configure the AWS CLI

To allow the CLI to interact with your AWS account, you must configure it with your IAM user credentials.
Run `aws configure` and provide the following details when prompted:

* **AWS Access Key ID:** (e.g., `AKIAIOSFODNN7EXAMPLE`)
* **AWS Secret Access Key:** (e.g., `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`)
* **Default region name:** `us-west-2`
* **Default output format:** `json`

### Task D: Interact with AWS Services (IAM Example)

Once configured, you can start managing resources.

* **List IAM Users:**
```bash
aws iam list-users

```


* **List Local Customer Managed Policies:**
```bash
aws iam list-policies --scope Local

```


* **Retrieve a specific Policy Document (Challenge Solution):**
By combining the Policy ARN and Version ID, you can download the actual JSON permissions into a local file:
```bash
aws iam get-policy-version --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/lab_policy --version-id v1 > lab_policy.json

```



## 4. Code Snippets Quick Reference

**Basic CLI Command Structure:**

```bash
aws <command> <subcommand> [options and parameters]

```

**Help Command:**
If you ever forget a command or its parameters, use the built-in help manual:

```bash
aws help
aws ec2 help
aws s3 ls help

```

## 5. Best Practices & Security

* **Never Hardcode Credentials:** Do not write your Access Keys directly into your scripts or code repositories (like GitHub).
* **Use IAM Roles for EC2:** In a real production environment, instead of running `aws configure` on an EC2 instance, you should attach an **IAM Role** to the instance. The AWS CLI will automatically use the role's temporary credentials, which is much more secure.
* **Keep Secrets Safe:** Treat your Secret Access Key like a password. If it is ever leaked, delete or deactivate the key immediately in the AWS Console.
* **Principle of Least Privilege:** Ensure the IAM User or Role configured with the CLI only has the permissions strictly necessary for its tasks.

---