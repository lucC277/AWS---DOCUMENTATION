# AWS Lambda & Serverless Computing

## 1. Overview & Key Features

AWS Lambda is a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers. You pay only for the compute time you consume.

In a typical serverless reporting architecture, Lambda can be triggered by a schedule, query a database (stored in a VPC), and send formatted results to an administrator via Amazon SNS.

**Key Features:**

* **No Servers to Manage:** AWS handles all the infrastructure, patching, and operating system maintenance.
* **Continuous Scaling:** Scales automatically by running code in response to each trigger.
* **Sub-second Metering:** You are charged for every millisecond your code executes and the number of times it is triggered.
* **Event-Driven:** Integrates natively with over 200 AWS services (e.g., EventBridge, S3, DynamoDB, SNS).

## 2. Core Concepts

* **Execution Role:** An IAM role attached to a Lambda function that grants it permission to interact with other AWS services (e.g., writing logs to CloudWatch, accessing a VPC, or reading from Parameter Store).
* **Lambda Layer:** A .zip file archive that contains libraries, a custom runtime, or other dependencies. Layers allow you to keep your deployment package small and share code (like `PyMySQL`) across multiple functions.
* **Environment Variables:** Key-value pairs that allow you to dynamically pass settings (like an SNS Topic ARN) to your function code without hardcoding them.
* **VPC Integration:** By default, Lambda runs in a secure AWS-managed VPC. To access private resources (like an RDS or EC2 database), you must configure the function to connect to your custom VPC.
* **EventBridge (CloudWatch Events):** A serverless event bus used to trigger Lambda functions based on a schedule (Cron job) or changes in your AWS environment.

## 3. Hands-On Guide: Building a Serverless Reporting Tool

### Task A: Prepare IAM Execution Roles

Before creating functions, ensure they have the necessary permissions:

1. **Reporting Function Role:** Needs `AWSLambdaBasicRunRole` (for logs), `AmazonSNSFullAccess`, `AmazonSSMReadOnlyAccess`, and permission to invoke other Lambda functions.
2. **Data Extractor Function Role:** Needs `AWSLambdaBasicRunRole` and `AWSLambdaVPCAccessRunRole` (to create network interfaces to access the database inside the VPC).

### Task B: Create a Lambda Layer (External Dependencies)

If your Python code requires external libraries (like `PyMySQL` to connect to a database), put them in a Layer.

1. Navigate to **Lambda** > **Layers** > **Create layer**.
2. Name it (e.g., `pymysqlLibrary`), upload the `.zip` file containing the library, and select the compatible runtime (e.g., `Python 3.9`).
3. Click **Create**.

### Task C: Deploy the Data Extractor Function (VPC Access)

1. Go to **Functions** > **Create function** (Author from scratch).
2. **Name:** `salesAnalysisReportDataExtractor` (Runtime: Python 3.9).
3. **Execution Role:** Attach the pre-created VPC-enabled IAM role.
4. **Add Layer:** In the function overview, click **Layers** > **Add a layer** > select your custom `pymysqlLibrary`.
5. **VPC Configuration:** Go to the **Configuration** tab > **VPC**. Select the VPC, Private/Public Subnets, and the Security Group that allows access to your database.
6. **Upload Code:** Upload your Python `.zip` deployment package.

### Task D: Set up Amazon SNS for Email Notifications

1. Go to the **Amazon SNS Dashboard** > **Topics** > **Create topic** (Standard).
2. Name the topic (e.g., `salesAnalysisReportTopic`).
3. Once created, click **Create subscription**, choose **Email** as the protocol, and enter your email address.
4. Confirm the subscription by clicking the link sent to your email inbox. Save the Topic ARN.

### Task E: Deploy the Reporting Function (via AWS CLI)

You can create functions directly using the AWS CLI.

1. Have your deployment package (`salesAnalysisReport-v2.zip`) ready locally.
2. Run the `create-function` command (see *Code Snippets* below).
3. **Set Environment Variable:** In the AWS Console, go to the new function's **Configuration** > **Environment variables**. Add `topicARN` as the Key and paste the SNS Topic ARN as the Value.

### Task F: Schedule the Execution with EventBridge

1. In the Reporting Function overview, click **Add trigger**.
2. Select **EventBridge (CloudWatch Events)**.
3. Choose **Create a new rule**, name it, and select **Schedule expression**.
4. Enter a Cron expression (e.g., `cron(0 20 ? * MON-SAT *)` to run at 8:00 PM UTC, Monday through Saturday).
5. Click **Add**.

## 4. Code Snippets & CLI Reference

**Create a Lambda Function via AWS CLI:**

```bash
aws lambda create-function \
--function-name salesAnalysisReport \
--runtime python3.9 \
--zip-file fileb://salesAnalysisReport-v2.zip \
--handler salesAnalysisReport.lambda_handler \
--region us-west-2 \
--role arn:aws:iam::123456789012:role/salesAnalysisReportRole

```

**Common CRON Expressions for EventBridge:**
AWS Cron expressions use UTC time and consist of 6 fields: `cron(Minutes Hours Day-of-month Month Day-of-week Year)`

* Every day at 8:00 PM UTC: `cron(0 20 * * ? *)`
* Monday to Saturday at 8:00 PM UTC: `cron(0 20 ? * MON-SAT *)`
* Every 5 minutes: `rate(5 minutes)`

## 5. Best Practices & Troubleshooting

* **Timeouts:** The default Lambda timeout is 3 seconds. If your function connects to a database or external API, you will likely need to increase this in the **General configuration** tab (max 15 minutes).
* **VPC Cold Starts:** Functions connected to a VPC used to suffer from heavy cold starts. While AWS has improved this natively, ensure your Security Groups strictly allow traffic between the Lambda and the DB port (e.g., 3306 for MySQL) to prevent connection timeouts.
* **Secrets Management:** Never hardcode database credentials in Lambda code. Store them in **AWS Systems Manager Parameter Store** or **AWS Secrets Manager**, and query them at runtime.
* **CloudWatch Logs:** Every `print()` statement in Python and all execution errors are automatically sent to **Amazon CloudWatch Logs**. If a test fails, checking the Log Group associated with your function is the first step in troubleshooting.

---