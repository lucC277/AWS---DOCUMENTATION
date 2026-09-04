Here is the documentation for **Infrastructure Monitoring, CloudWatch Agent, Logs, Events, and AWS Config**, 
---

# Infrastructure Monitoring: CloudWatch, CloudWatch Agent, and AWS Config

## 1. Overview & Key Features

Effective monitoring and auditing are critical for maintaining high availability, performance, and security compliance in cloud environments. AWS provides native toolsets to capture system metrics, stream application logs, trigger real-time operational events, and continuously evaluate resource configurations.

**Key Features:**

* **Granular Metrics & Logs:** Use the CloudWatch Agent to collect OS-level metrics (CPU, memory, disk utilization) and application/system logs from EC2 instances.
* **Metric Filters & Alarms:** Turn raw log text into searchable metrics and trigger automated alarms when performance thresholds or error rates are crossed.
* **Event-Driven Automation:** Capture near-real-time state changes using CloudWatch Events (EventBridge) to trigger notifications or workflows.
* **Continuous Compliance Auditing:** Use AWS Config to evaluate resource configurations against organizational standards (such as required tagging and unattached volume checks).

## 2. Core Concepts

* **CloudWatch Agent:** Software installed on EC2 or on-premises servers that streams custom system metrics and log files to Amazon CloudWatch.
* **Parameter Store (Systems Manager):** Secure, hierarchical storage used here to store the JSON configuration file for the CloudWatch Agent.
* **Metric Filter:** A pattern-matching rule applied to CloudWatch Logs that converts matching log text (e.g., `404 status code`) into a numeric metric.
* **CloudWatch Events / EventBridge:** Streams operational changes in AWS resources and routes them to targets like Amazon SNS.
* **AWS Config Rules:** Evaluates whether AWS resources comply with predefined configurations (e.g., ensuring all resources possess a specific tag key).

## 3. Hands-On Guide: Implementing Monitoring & Compliance

### Task A: Deploying and Configuring the CloudWatch Agent via Systems Manager

To collect internal operating system metrics and logs from an EC2 instance:

1. **Install the Agent Package:**
* Navigate to **Systems Manager** > **Run Command** > **Run command**.
* Select `AWS-ConfigureAWSPackage`.
* *Action:* `Install`, *Name:* `AmazonCloudWatchAgent`, *Version:* `latest`.
* Select your target EC2 instance (`Web Server`) and execute.


2. **Store the Agent Configuration:**
* Go to **Parameter Store** > **Create parameter**.
* *Name:* `Monitor-Web-Server`
* *Value:* Input the JSON configuration defining the log paths (`/var/log/httpd/access_log`) and system metrics (`cpu`, `disk`, `mem`).


3. **Start the Agent:**
* Run another command using `AmazonCloudWatch-ManageAgent`.
* *Action:* `configure`, *Mode:* `ec2`, *Configuration Source:* `ssm`, *Configuration Location:* `Monitor-Web-Server`, *Restart:* `yes`.



### Task B: Monitoring Application Logs & Creating Metric Filters

1. **Generate Log Data:** Access your web server's public IP and append an invalid path (e.g., `/start`) to trigger a `404 Not Found` error in the Apache access logs.
2. **View Logs in CloudWatch:** Navigate to **CloudWatch** > **Log groups** to inspect `HttpAccessLog` and `HttpErrorLog`.
3. **Create a Metric Filter:**
* Select `HttpAccessLog` > **Actions** > **Create metric filter**.
* *Filter Pattern:* `[ip, id, user, timestamp, request, status_code=404, size]`
* *Metric Details:* Namespace: `LogMetrics`, Name: `404Errors`, Value: `1`.


4. **Create an Alarm:**
* Create an alarm based on the `404Errors` metric.
* *Condition:* Greater than or equal to `5` occurrences in a `1-minute` period.
* *Notification:* Send alerts to an Amazon SNS topic linked to your email address.



### Task C: Real-Time Event Notifications

Configure alerts to fire immediately when critical infrastructure changes occur (e.g., an instance stops or terminates).

1. Navigate to **CloudWatch** > **Events** > **Rules** > **Create rule**.
2. *Event Pattern:* AWS Service > **EC2** > EC2 Instance State-change Notification > Specific state(s): `stopped` and `terminated`.
3. *Targets:* Route event to an **SNS topic** (e.g., `Default_CloudWatch_Alarms_Topic`).

### Task D: Tracking Infrastructure Compliance with AWS Config

Ensure resources adhere to operational rules and tagging guidelines.

1. Navigate to the **AWS Config** console and complete the initial setup wizard if prompted.
2. Go to **Rules** > **Add rule**.
3. **Required Tags Rule:** Search for `required-tags`, set parameter `tag1Key` to `project`. This evaluates whether resources are properly tagged.
4. **Unattached Volumes Rule:** Add the `ec2-volume-inuse-check` rule to identify unattached EBS volumes wasting costs.
5. View evaluation results under **Resources in scope** to distinguish compliant versus non-compliant infrastructure.

## 4. Best Practices & Security

* **Centralized Logging:** Always ship application and system logs from EC2 instances to CloudWatch Logs. This ensures logs are safely preserved even if the underlying instance crashes or is terminated.
* **Least Privilege IAM Roles:** Ensure EC2 instances running the CloudWatch Agent have an IAM role attached with the `CloudWatchAgentServerPolicy` managed policy.
* **Proactive Compliance:** Use AWS Config rules alongside automated remediation (via AWS Systems Manager Automation or Lambda) to instantly fix non-compliant resources (e.g., automatically terminating unencrypted volumes or adding missing tags).

---