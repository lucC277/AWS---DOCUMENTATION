# Amazon Route 53: Failover Routing & Health Checks

## 1. Overview & Key Features

Amazon Route 53 is a highly available and scalable cloud Domain Name System (DNS) web service. It is designed to give developers and businesses an extremely reliable and cost-effective way to route end users to internet applications.

In an Active-Passive architecture, you can use Route 53 **Failover Routing** to automatically detect an outage of your primary web server and redirect traffic to a standby (secondary) server in a different Availability Zone or Region.

**Key Features:**

* **Global DNS Resolution:** Translates human-readable names (like `[www.example.com](https://www.example.com)`) into numeric IP addresses.
* **Health Checks:** Monitors the health and performance of your web applications, web servers, and other resources.
* **Traffic Management:** Supports multiple routing policies (Simple, Failover, Geolocation, Latency, Weighted, and Multi-Value Answer).
* **Automated Alerts:** Integrates with Amazon SNS to send email or SMS notifications when an endpoint goes down.

## 2. Core Concepts

* **Hosted Zone:** A container that holds information about how you want to route traffic on the internet for a specific domain.
* **A Record (Address Record):** A DNS record that maps a domain name to an IPv4 address.
* **Failover Routing Policy:** Used to configure active-passive failover. Route 53 responds to DNS queries using the primary record as long as its health check is passing, and falls back to the secondary record if the primary is unhealthy.
* **TTL (Time to Live):** The amount of time (in seconds) that DNS resolvers should cache the record. A low TTL (e.g., 15-60 seconds) is crucial for fast failovers.
* **Amazon SNS (Simple Notification Service):** A messaging service used here to trigger email alerts when a Route 53 Health Check fails.

## 3. Hands-On Guide: Configuring Active-Passive Failover

### Task A: Create a Route 53 Health Check

Before configuring failover, Route 53 needs a mechanism to know if the primary server is functioning.

1. Navigate to the **Route 53 Dashboard** and select **Health checks**.
2. Click **Create health check**.
3. **Configuration:**
* **Name:** `Primary-Website-Health`
* **Monitor:** `Endpoint` (Specify by IP address).
* **IP Address:** Enter the Public IPv4 address of your Primary EC2 instance.
* **Advanced:** Set *Request interval* to **Fast (10 seconds)** and *Failure threshold* to **2**. This ensures Route 53 detects failures quickly (within ~20 seconds).


4. **Notifications:** Choose **Yes** to Create alarm, select **New SNS topic**, and provide your email address to receive downtime alerts. (Remember to confirm the AWS SNS subscription in your email inbox).

### Task B: Create the Primary DNS Record

1. In Route 53, navigate to **Hosted zones** and select your domain name.
2. Click **Create record**.
* **Record name:** `www`
* **Record type:** `A - Routes traffic to an IPv4 address`
* **Value:** `[Primary EC2 Instance IP]`
* **TTL:** `15` seconds (Low TTL forces browsers to query Route 53 more frequently, enabling rapid failover).
* **Routing policy:** `Failover`
* **Failover record type:** `Primary`
* **Health check ID:** Select `Primary-Website-Health`.
* **Record ID:** `FailoverPrimary`


3. Save the record.

### Task C: Create the Secondary DNS Record

Now, map the secondary/standby server.

1. Click **Create record** again in the same Hosted Zone.
* **Record name:** `www` (Must match the primary record exactly).
* **Record type:** `A`
* **Value:** `[Secondary EC2 Instance IP]`
* **TTL:** `15` seconds
* **Routing policy:** `Failover`
* **Failover record type:** `Secondary`
* **Health check ID:** Leave empty (the secondary instance assumes traffic passively).
* **Record ID:** `FailoverSecondary`


2. Save the record.

### Task D: Test and Verify Failover

1. Open a browser and navigate to your domain (e.g., `[www.your-domain.com/cafe](https://www.your-domain.com/cafe)`). Note which Availability Zone is serving the traffic (it should be the Primary).
2. Go to the **EC2 Console**, select your Primary EC2 Instance, and **Stop** it to simulate a server crash.
3. Wait approximately 1-2 minutes:
* You will receive an **Alarm Email** from SNS notifying you that the endpoint is unhealthy.
* The Route 53 Health Check status will change to **Unhealthy**.


4. Refresh your browser page. The DNS resolution will have automatically switched, and the page will now be served from the Secondary EC2 instance in the other Availability Zone.

## 4. Configuration Quick Reference

| Record Name | Type | Value (IP) | TTL | Routing Policy | Failover Type | Health Check Attached? |
| --- | --- | --- | --- | --- | --- | --- |
| `www` | A | Primary IP | 15s | Failover | Primary | Yes |
| `www` | A | Secondary IP | 15s | Failover | Secondary | No |

## 5. Best Practices & Security

* **TTL Optimization:** For Failover and Weighted routing, always use a low TTL (like 60 seconds or less). If you use a standard TTL of 24 hours, users' internet providers will cache the broken IP address for a day, rendering the failover useless.
* **Health Check Costs:** "Fast" health checks (every 10 seconds) cost more than standard health checks (every 30 seconds). Use Fast checks only for critical mission workloads.
* **Monitor the Secondary:** In production, it is highly recommended to also attach a Health Check to the Secondary record. This prevents Route 53 from routing traffic to a standby server that might also be offline.
* **Beyond IP Addresses:** Route 53 can route traffic directly to other AWS resources (like Application Load Balancers, CloudFront distributions, or S3 Website endpoints) using `Alias` records, which are free of charge for DNS queries.

---