
```markdown
# AWS Cloud SOC Automation Lab

## Overview

I designed and implemented a **real-time cloud security automation system** on AWS to detect unauthorized access to sensitive S3 buckets, immediately contain compromised users, and notify security teams. The goal was to replicate a real-world SOC workflow, demonstrating hands-on skills in cloud security, serverless automation, and incident response.

This project showcases how **CloudTrail, EventBridge, Lambda, IAM, and SNS** can be orchestrated to create a fully automated incident response pipeline.

---

## Motivation

As part of learning cloud security and SOC automation, I wanted to create a lab where I could **simulate unauthorized access attempts** and automatically respond in real time. The key objectives were:

- Detect when a non-root IAM user accessed a sensitive bucket or file.  
- Automatically restrict any further actions by the user.  
- Send an immediate alert to a SOC team.  
- Log all events for audit and review.  

The project aimed to demonstrate both **technical skills** and an understanding of **security operations best practices**.

---

## Architecture

The system followed a simple but effective workflow:

```

User attempts access to S3 bucket/file
│
▼
AWS CloudTrail logs the API event
│
▼
Amazon EventBridge filters for suspicious activity
│
▼
AWS Lambda executes automated response
│
├─ Attach DenyAll IAM policy to user
└─ Send SOC alert via SNS email

````

**Services Used:**

- **AWS CloudTrail** – Captured all API activity including S3 object and bucket access.  
- **Amazon EventBridge** – Matched events for specific buckets and actions, triggering automation.  
- **AWS Lambda** – Contained the user and sent alerts.  
- **AWS IAM** – Managed user permissions and applied DenyAll policy.  
- **Amazon SNS** – Delivered alerts to the SOC team via email.  

---

## Implementation Steps

### 1. Setup Sensitive S3 Bucket

- Created an S3 bucket: `sensitive-company-data`  
- Uploaded confidential files such as `payroll.csv` and `financial-reports.xlsx`  
- Configured bucket permissions to simulate a production environment  

### 2. Enable CloudTrail for S3 Data Events

- Created a trail named `security-monitor-trail`  
- Enabled **management and data events** for all regions  
- Selected **S3 data events** for the sensitive bucket  
- Confirmed that **GetObject, PutObject, DeleteObject, HeadBucket, ListObjects** were all logged  

### 3. Create DenyAll IAM Policy

- Created a policy that explicitly **denies all actions**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Deny", "Action": "*", "Resource": "*" }
  ]
}
````

* Named it `Emergency-DenyAll`
* Verified that attaching it to a user would override any existing permissions, including administrator privileges

### 4. Create Lambda Function

* Runtime: Python 3.11
* Execution role had permissions:

  * `iam:AttachUserPolicy` – to apply DenyAll
  * `sns:Publish` – to send alerts

**Lambda Function Highlights:**

```python
import json
import boto3

iam = boto3.client('iam')
sns = boto3.client('sns')

DENY_POLICY_ARN = "<your-DenyAll-policy-ARN>"
SNS_TOPIC_ARN = "<your-SNS-topic-ARN>"

def lambda_handler(event, context):
    # Extract event details
    user = event["detail"]["userIdentity"]["userName"]
    action = event["detail"]["eventName"]
    bucket = event["detail"]["requestParameters"]["bucketName"]
    time = event["detail"]["eventTime"]
    source_ip = event["detail"].get("sourceIPAddress", "Unknown")

    # Contain user
    iam.attach_user_policy(UserName=user, PolicyArn=DENY_POLICY_ARN)

    # Prepare alert
    message = f"""
SECURITY INCIDENT DETECTED

User: {user}
Action: {action}
Bucket: {bucket}
Time: {time}
Source IP: {source_ip}

Automated Response:
DenyAll policy applied to user. Account locked pending investigation.
"""
    # Send SOC alert
    sns.publish(TopicArn=SNS_TOPIC_ARN, Subject="SOC Alert: Unauthorized S3 Access", Message=message)
```

### 5. Configure EventBridge Rule

* Created a rule to trigger Lambda for events matching:

  * `eventSource = "s3.amazonaws.com"`
  * `eventName = ["GetObject", "PutObject", "DeleteObject", "HeadBucket", "ListObjects"]`
  * `bucketName = "sensitive-company-data"`
  * `userIdentity.type = "IAMUser"`

### 6. Setup SNS for SOC Alerts

* Created SNS topic: `soc-security-alerts`
* Subscribed my email and confirmed subscription
* Lambda function published alerts to SNS for every detected event

---

## Testing

1. Created a test IAM user with normal read/write permissions
2. Attempted to access the sensitive bucket:

   * `GetObject`
   * `ListObjects`
   * `HeadBucket`

**Observed Behavior:**

* Lambda triggered on every relevant API call
* User was instantly restricted by DenyAll policy
* Email alerts sent to inbox with all details: user, action, bucket, timestamp, and automated response

---

## Lessons Learned

* Built a **complete automated SOC workflow** in AWS
* Learned the importance of **explicit deny** in IAM and post-event containment
* Gained practical experience in **serverless automation, event-driven architecture, and security alerting**
* Discovered the difference between **detection vs prevention**, and ways to enhance alerts with Slack, IP info, and event aggregation

---

## Potential Enhancements

* Aggregate multiple API calls into a single alert to reduce noise
* Add Slack/Teams or SMS alerts for faster response
* Include **source IP, user agent, and severity tags**
* Implement **preemptive bucket policies** to block access before it happens

---

## Outcome

By the end of this project, I had a **fully functional AWS SOC automation lab** that could:

* Detect unauthorized access in real-time
* Automatically contain compromised users
* Alert security teams immediately
* Log every event for audits

This project demonstrates **hands-on expertise in cloud security, serverless automation, and incident response workflows**, making it strong for SOC, cloud security, and DevSecOps portfolios.

```
