Aqui está a documentação para **AWS CloudFormation (Infraestrutura como Código, Templates YAML, Gerenciamento de Stacks e Atualizações)**, 
---

# AWS CloudFormation: Infrastructure as Code (IaC) & Stack Management

## 1. Overview & Key Features

**AWS CloudFormation** gives developers and administrators an easy way to create and manage a collection of related AWS resources, provisioning and updating them in an orderly and predictable fashion. By defining infrastructure in text templates using JSON or YAML, organizations achieve consistency, repeatability, and version control (Infrastructure as Code - IaC).

**Key Features:**

* **Declarative Templates:** Define what resources you want to deploy, and CloudFormation automatically determines the optimal provisioning order and dependencies.
* **Automated Rollbacks:** If a resource creation or update fails, CloudFormation rolls back the stack, safely removing any partially created resources.
* **Change Sets:** Preview how proposed changes to a stack (like updating a resource property) will impact running resources before applying them.
* **Dynamic Parameter Resolution:** Integrate with Systems Manager Parameter Store to fetch region-specific AMIs or configurations dynamically.

## 2. Core Concepts

* **Template:** A formatted text file (YAML or JSON) structured into sections: `Parameters`, `Mappings`, `Conditions`, `Resources`, and `Outputs`.
* **Stack:** The collection of AWS resources created and managed as a single unit when a CloudFormation template is deployed.
* **Resources Section:** The only mandatory section in a template, defining the actual AWS components to deploy (e.g., `AWS::EC2::VPC`, `AWS::S3::Bucket`).
* **Intrinsic Functions (`!Ref`, `!GetAtt`, etc.):** Built-in CloudFormation functions used to assign values dynamically, reference other resources within the template, or fetch attributes.

---

## 3. Hands-On Guide: Writing Templates & Managing Stacks

### Task A: Deploying a Basic CloudFormation Stack

1. **Prepare the Template (`task1.yaml`):**
The base template defines parameters for IP ranges, provisions a custom VPC, and sets up a default Security Group.
2. **Deploy via Console:**
* Navigate to the **CloudFormation Console** > **Create stack** > **Upload a template file**.
* Upload `task1.yaml`, assign a Stack name (e.g., `Lab`), and proceed through the wizard without modifying default parameters.
* Review the **Events** and **Resources** tabs to watch CloudFormation build the infrastructure dependencies sequentially.



---

### Task B: Modifying a Stack (Adding an S3 Bucket)

Adding resources to an existing stack is fast and safe because CloudFormation updates only the delta changes.

1. **Edit the Template:**
Open your YAML file and add a minimal S3 bucket resource under the `Resources:` header:
```yaml
Resources:
  # (Existing VPC and Security Group definitions...)

  MyS3Bucket:
    Type: AWS::S3::Bucket

```


*(Note: YAML requires strict two-space indentation).*
2. **Update the Stack:**
* In the CloudFormation console, select your stack (`Lab`), and click **Update**.
* Choose **Replace current template** > **Upload a template file** (select your modified YAML file).
* Review the **Change Set** preview to verify that CloudFormation will safely add the S3 bucket without modifying existing resources. Click **Update stack**.



---

### Task C: Referencing Resources & Dynamic Parameters (Adding an EC2 Instance)

To deploy complex resources like an EC2 instance, you must tie together parameters, dynamic AMIs, and cross-resource references (`!Ref`).

1. **Add an SSM Parameter for the Latest AMI:**
Add this block to the `Parameters` section so CloudFormation automatically fetches the latest Amazon Linux 2 AMI ID for the active region:
```yaml
Parameters:
  AmazonLinuxAMIID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

```


2. **Define the EC2 Instance Resource:**
Use `!Ref` to reference previously declared parameters and resources (like security groups and subnets):
```yaml
AppServer:
  Type: AWS::EC2::Instance
  Properties:
    ImageId: !Ref AmazonLinuxAMIID
    InstanceType: t3.micro
    SecurityGroupIds:
      - !Ref AppSecurityGroup
    SubnetId: !Ref PublicSubnet
    Tags:
      - Key: Name
        Value: App Server

```


3. **Update the Stack:** Upload the completed template (`task3.yaml`) to update the stack once more.

---

### Task D: Deleting a Stack

When a stack is deleted, CloudFormation automatically tears down all associated resources in the correct reverse dependency order.

* In the CloudFormation console, select the `Lab` stack, click **Delete**, and confirm. The entire architecture (VPC, Subnets, S3 Bucket, EC2 Instance, Security Groups) is removed cleanly.

## 4. Best Practices & Security

* **Idempotency:** Write templates that can be repeatedly deployed without unexpected errors or duplicated non-unique resource names.
* **Use Parameter Store for AMIs:** Avoid hardcoding AMI IDs into templates. Always query the AWS Systems Manager Parameter Store public paths for standard OS images so your templates work seamlessly across multiple regions.
* **Version Control:** Store all CloudFormation templates in a Git repository (like GitHub) to track infrastructure changes, collaborate with team members, and roll back bad configurations if necessary.

---