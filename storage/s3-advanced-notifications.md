Aqui está a documentação para **Amazon S3 (Gerenciamento Avançado, Políticas de Acesso Granulares, Notificações de Eventos via SNS e Integração com CLI)**
---

# Amazon S3: Advanced Sharing, Granular IAM Policies & Event Notifications

## 1. Overview & Key Features

Beyond simple file storage, Amazon S3 enables secure multi-user collaboration and automated event-driven workflows. By combining **IAM Policies**, **Prefix-Level Permissions**, **Amazon SNS**, and **S3 Event Notifications**, administrators can grant external partners least-privilege access to specific folders while maintaining full auditing and real-time alerting.

**Key Features:**

* **Granular Prefix Permissions:** Restrict external users (such as media vendors or partners) to specific subfolders (e.g., `/images/*`) inside an S3 bucket.
* **Event-Driven Workflows:** Automatically trigger alerts or automations when objects are created (`ObjectCreated`) or removed (`ObjectRemoved`).
* **SNS Integration:** Publish real-time JSON notifications to an Amazon Simple Notification Service topic to alert administrators of bucket modifications.
* **API Control via CLI:** Use `s3api` commands to manage bucket policies, access control lists, and event notification configurations programmatically.

## 2. Core Concepts

* **Prefix-Level Access:** A security practice where IAM policies limit user actions to specific folder paths within a bucket rather than the entire container.
* **S3 Event Notifications:** A feature that configures S3 to send a notification message when specific events occur (e.g., uploads, deletions).
* **Topic Configuration:** Maps specific S3 event types and prefix filters to an SNS Topic ARN.
* **SNS Access Policy:** A resource-based policy attached to an SNS topic that explicitly allows the S3 service principal (`s3.amazonaws.com`) to publish messages to it.

## 3. Hands-On Guide: Configuring Secure File Sharing & Alerts

### Task A: Create and Initialize the S3 Bucket

1. **Create the Bucket:**
```bash
aws s3 mb s3://<cafe-xxxnnn> --region 'us-west-2'

```


2. **Sync Initial Content (Images folder):**
Upload baseline assets into a specific prefix directory:
```bash
aws s3 sync ~/initial-images/ s3://<cafe-xxxnnn>/images

```



### Task B: Enforcing Prefix-Level IAM Permissions

To allow external media users (`mediacouser`) to upload or delete images without exposing the entire bucket or administrative settings:

* **Policy Structure:** The IAM policy uses conditions and resource ARNs scoped specifically to the prefix:
`arn:aws:s3:::cafe-*/images/*`
* **Allowed Operations:** `GetObject`, `PutObject`, and `DeleteObject` (Read, Write, Delete).
* **Blocked Operations:** Changing bucket-level permissions or accessing the bucket root are explicitly denied by omission or restriction, returning an `AccessDenied` error if attempted.

### Task C: Configuring Event Notifications via Amazon SNS

Automate email alerts whenever files are added or removed from the `images/` directory.

1. **Create an SNS Topic:**
Go to the **SNS Console** > **Topics** > **Create topic** (Standard), name it `s3NotificationTopic`, and copy its **ARN**. Subscribe your email address and confirm the subscription via your inbox.
2. **Grant S3 Permission to Publish to SNS:**
Attach an **Access Policy** to your SNS topic allowing the S3 service to publish messages:
```json
{
  "Version": "2008-10-17",
  "Id": "S3PublishPolicy",
  "Statement": [
    {
      "Sid": "AllowPublishFromS3",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "<ARN of s3NotificationTopic>",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:*:*:<cafe-xxxnnn>"
        }
      }
    }
  ]
}

```


3. **Define the S3 Event Notification Configuration (`s3EventNotification.json`):**
Create a local JSON file mapping events and prefixes:
```json
{
  "TopicConfigurations": [
    {
      "TopicArn": "<ARN of s3NotificationTopic>",
      "Events": ["s3:ObjectCreated:*", "s3:ObjectRemoved:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "images/"
            }
          ]
        }
      }
    }
  ]
}

```


4. **Apply the Configuration to the Bucket:**
Use the `s3api` command to push the notification settings:
```bash
aws s3api put-bucket-notification-configuration \
    --bucket <cafe-xxxnnn> \
    --notification-configuration file://s3EventNotification.json

```


*Note: Applying this triggers an automated `s3:TestEvent` sent to your email inbox to verify connectivity.*

### Task D: Testing Uploads, Deletions, and Notifications

Configure your CLI with the external user's (`mediacouser`) access keys to test workflows:

1. **Upload an Object (Triggers `ObjectCreated:Put`):**
```bash
aws s3api put-object --bucket <cafe-xxxnnn> \
    --key images/Caramel-Delight.jpg \
    --body ~/new-images/Caramel-Delight.jpg

```


*Result:* An email notification is instantly dispatched detailing the upload.
2. **Read an Object (No Notification):**
```bash
aws s3api get-object --bucket <cafe-xxxnnn> --key images/Donuts.jpg Donuts.jpg

```


*Result:* Downloads successfully, but **no email is sent** because `GetObject` is excluded from the notification rules.
3. **Delete an Object (Triggers `ObjectRemoved:Delete`):**
```bash
aws s3api delete-object --bucket <cafe-xxxnnn> --key images/Strawberry-Tarts.jpg

```


*Result:* An email notification is sent confirming the object removal.
4. **Unauthorized Action Test:**
Attempting actions outside permitted boundaries (such as modifying object ACLs: `--acl public-read`) correctly throws an `AccessDenied` exception.

## 4. Best Practices & Security

* **Scoped IAM Policies:** Always restrict partner or automated service accounts to specific prefixes (e.g., `bucket-name/folder/`) rather than granting broad bucket-level permissions.
* **Filter Rules:** Use S3 notification filters (`prefix` and `suffix`) to avoid flooding notification endpoints (like SNS or SQS) with irrelevant events from temporary folders.
* **SNS Topic Policies:** Always validate the `aws:SourceArn` condition in your SNS access policies to prevent unauthorized cross-service confused deputy vulnerabilities.

---