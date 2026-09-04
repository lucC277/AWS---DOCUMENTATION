Aqui está a documentação do **AWS Identity and Access Management (IAM)** 
---

# AWS Identity and Access Management (IAM)

## 1. Overview & Key Features

AWS Identity and Access Management (IAM) is a web service that helps you securely control access to AWS resources. You use IAM to control who is authenticated (signed in) and authorized (has permissions) to use resources.

**Key Features:**

* **Centralized Access Control:** Manage users, security credentials, and permissions from a single pane of glass.
* **Shared Access:** Grant other people permission to administer and use resources in your AWS account without sharing your password or access key.
* **Granular Permissions:** Apply different permissions to different people for different resources (Principle of Least Privilege).
* **Identity Federation:** Allow users who already have passwords elsewhere (e.g., corporate network, Google) to access your AWS account.

## 2. Core Concepts

* **IAM User:** An entity that you create in AWS to represent the person or application that uses it to interact with AWS.
* **IAM Group (User Group):** A collection of IAM users. Groups let you specify permissions for multiple users, making it easier to manage those permissions.
* **IAM Role:** An identity with permission policies that determine what the identity can and cannot do in AWS. Unlike a user, a role is not uniquely associated with one person and is assumed temporarily.
* **IAM Policy:** A document (typically in JSON format) that defines permissions.
* *Managed Policies:* Pre-built policies (by AWS or administrators) that can be attached to multiple users/groups.
* *Inline Policies:* Custom policies embedded directly into a single user, group, or role.


* **Policy Structure:** Built around three main elements:
* `Effect`: Whether to **Allow** or **Deny** access.
* `Action`: The specific API calls allowed (e.g., `ec2:StopInstances`).
* `Resource`: The specific entities covered by the rule (e.g., an S3 bucket ARN, or `*` for all resources).



## 3. Hands-On Guide: Managing Access and Permissions

### Task A: Configure a Custom Password Policy

To enhance account security, you can enforce strict password requirements for all IAM users.

1. Navigate to the **IAM Dashboard**.
2. Go to **Account settings**.
3. Choose **Change password policy**.
4. Enforce rules such as: Minimum length (e.g., 10 characters), require symbols/numbers/uppercase letters, and set password expiration (e.g., 90 days).
5. Save changes to apply this account-wide.

### Task B: Manage Users and User Groups

Instead of assigning permissions directly to users, assign them to groups based on job functions.

1. **Explore Groups:** In the left pane, select **User groups**. You might have groups like `S3-Support` (Read-only S3 access), `EC2-Support` (Read-only EC2 access), and `EC2-Admin` (Full EC2 control).
2. **Assign Users:**
* Select a group (e.g., `S3-Support`).
* Go to the **Users** tab and click **Add users**.
* Select the respective user (e.g., `user-1`) and confirm.


3. **Repeat** for other roles (e.g., add `user-2` to `EC2-Support` and `user-3` to `EC2-Admin`).

### Task C: Test User Permissions via Sign-in URL

Always verify that the policies behave exactly as expected.

1. Go to the **IAM Dashboard** and copy the **Sign-in URL for IAM users** (e.g., `https://<account-id>[.signin.aws.amazon.com/console](https://.signin.aws.amazon.com/console)`).
2. Open a private/incognito browser window.
3. **Test Support Role (S3-Support):** Sign in as `user-1`. Verify you can browse S3 buckets. Navigate to EC2 and confirm you get an *Unauthorized* error.
4. **Test Read-Only Role (EC2-Support):** Sign out and sign in as `user-2`. Navigate to EC2 and view instances. Select an instance and attempt to **Stop** it. This will fail with an authorization error, proving the Read-Only policy works.
5. **Test Admin Role (EC2-Admin):** Sign out and sign in as `user-3`. Navigate to EC2, select an instance, and attempt to **Stop** it. The action will succeed.

## 4. Code Snippets / Policy Examples

**Example of an IAM Policy (JSON):**
This is a standard structure of an IAM policy. The one below represents a read-only support role for EC2 (similar to what `user-2` had).

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeImages",
                "ec2:DescribeTags",
                "ec2:DescribeSnapshots"
            ],
            "Resource": "*"
        }
    ]
}

```

## 5. Best Practices & Security

* **Use Groups for Permissions:** Never assign permissions directly to an IAM user. Add users to groups and apply policies to the groups.
* **Principle of Least Privilege:** Start with a minimum set of permissions and grant additional permissions only as necessary.
* **Enable MFA (Multi-Factor Authentication):** Always require MFA for privileged accounts, especially the root account.
* **Regularly Rotate Credentials:** Enforce password expiration and rotate API access keys frequently.
* **Do Not Use Root for Daily Tasks:** The AWS account root user has unrestricted access. Create an IAM admin user for everyday management and lock away the root credentials.

---