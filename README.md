# AWS-GuardDuty-to-Splunk-Integration-End-to-End-SIEM-Pipeline-
This project demonstrates how to build an end-to-end security monitoring pipeline by integrating AWS GuardDuty with Splunk using AWS services.

## Task 1: Enable GuardDuty
**1.** Log in to AWS Console

**2.** Navigate to GuardDuty

**3.** Click Get Started or Enable GuardDuty (if not already enabled)

**4.** Select your region and enable the service

**5.** Generate sample findings for testing:

- Go to Settings → Sample findings
- Click Generate sample findings
- This creates test data to verify your pipeline

## Task 2: Create SNS Topic
**1.** Go to AWS Console → SNS (Simple Notification Service)

**2.** Click Topics → Create topic

**3.** Configure the topic:

- Type: Standard
- Name: ```gd-findings-topic```
- Display name (optional): ```GuardDuty Findings```

**4.** Click Create topic

**5.** Copy the Topic ARN - you'll need this for EventBridge and SQS

**Example ARN:**  ```arn:aws:sns:us-east-1:123456789012:gd-findings-topic```

## Task 3: Create SQS Queue
**1.** Go to AWS Console → SQS (Simple Queue Service)

**2.** Click Create queue

**3.** Configure the queue:

- Type: Standard (not FIFO)
- Name: ```gd-findings-queue```
- Visibility timeout: 30 seconds (default is fine)
- Message retention period: 4 days (default)
- Maximum message size: 256 KB (default)

**4.** Keep all other settings as default

**5.** Click Create queue

**6.** Copy the Queue ARN and Queue URL - you'll need both

**Example Queue ARN:**  ```arn:aws:sqs:us-east-1:123456789012:gd-findings-queue```

## Task 4: Subscribe SQS Queue to SNS Topic
**1.** Go to SNS → Topics → Click on gd-findings-topic

**2.** Go to Subscriptions tab

**3.** Click Create subscription

**4.** Configure:

- **Protocol:**  Amazon SQS
- **Endpoint:**  Paste your SQS Queue ARN (from Step 3)

**5.** Click Create subscription

**6.** Status should show Confirmed immediately

## Task 5: Create EventBridge Rule
EventBridge will capture GuardDuty findings and send them to SNS.

**1.** Go to AWS Console → EventBridge

**2.** Click Rules → Create rule

**3.** Configure the rule:

- Name: ```guardduty-to-sns```

- Description (optional): ```Route GuardDuty findings to SNS```

- Event bus: default

**4.** Click Next

### Build Event Pattern
**1.** Under Event source, select AWS events or EventBridge partner events

**2.** Creation method: Select Use pattern form

**3.** Configure the event pattern:

- Event source: AWS services

- AWS service: Select GuardDuty

- Event type: Select GuardDuty Finding

AWS will automatically generate this event pattern for you:
```
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"]
}
```
- Click Next

### Select Target
**1.** Target types: AWS service

**2.** Select a target: SNS topic

**3.** Topic: Select ```gd-findings-topic``` (from the dropdown)

**4.** Click Next

**5.** (Optional) Add tags if needed, then click Next

**6.** Review your configuration and click Create rule

**7.** Ensure the rule State shows as Enabled

## Task 6: Create IAM User for Splunk
**1.** Go to AWS Console → IAM → Users

**2.** Click Create user

**3.** Username: ```splunk-guardduty-reader```

**4.** Select Provide user access to AWS Management Console - Optional (not needed)

**5.** Click Next

**6.** Skip adding permissions for now (we'll create a custom policy)

**7.** Click Create user

**8.** Create access key:

- Go to the user → Security credentials tab

- Click Create access key

- Select Application running outside AWS

- Click Next → Create access key

**9.** Download and save the Access Key ID and Secret Access Key securely

## Task 7: Create IAM Policy for Splunk
**1.** Go to IAM → Policies → Create policy

**2.** Click JSON tab

**3.** Paste the following policy (replace placeholders with your values):
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListQueues",
      "Effect": "Allow",
      "Action": [
        "sqs:ListQueues"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowSQSAccessForSplunk",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:ChangeMessageVisibility",
        "sqs:ListQueueTags"
      ],
      "Resource": "arn:aws:sqs:<REGION>:<ACCOUNT_ID>:gd-findings-queue"
    }
  ]
}
```

**Replace:**

- ```<REGION>:``` Your AWS region (e.g., ```ap-south-1```)

- ```<ACCOUNT_ID>:``` Your 12-digit AWS account ID

- ```gd-findings-queue```: Your queue name if different

**1.** Click Next

**2.** **Policy name:** ```SplunkGuardDutySQSReadPolicy```

**3.** Click Create policy

### Attach Policy to Splunk User
**1.** Go to IAM → Users → Select ```splunk-guardduty-reader```

**2.** Click Add permissions → Attach policies directly

**3.** Search for and select ```SplunkGuardDutySQSReadPolicy```

**4.** Click Add permissions

## Task 8: Configure Splunk Add-on for AWS

### 8.1 Install the Add-on (if not already installed)
**1.** In Splunk, go to Apps → Find More Apps

**2.** Search for Splunk Add-on for Amazon Web Services

**3.** Install the add-on

**4.** Restart Splunk if prompted

### 8.2 Add AWS Account
**1.** Go to Splunk Add-on for AWS → Configuration

**2.** Click Account tab

**3.** Click Add

**4.** Enter:

- Account Name: ```guardduty-sqs-account```

- AWS Access Key ID: (from Step 7)

- AWS Secret Access Key: (from Step 7)

**5.** Click Add

### 8.3 Create SQS Input
**1.** Go to Inputs tab

**2.** Click Create New Input → SQS-Based S3 or Generic SQS Input

**3.** Configure the input:

- Name: ```guardduty-sqs-input```

- AWS Account: Select ```guardduty-sqs-account```

- AWS Region: Select your region

- SQS Queue: Select or paste ```gd-findings-queue```

- Sourcetype: ```aws:guardduty``` (or ```aws:guardduty:notification```)

- Index: ```security``` (or create a dedicated index like ```aws_guardduty```)

- Interval: 30 seconds (default polling interval)

**4.** Click Add

## Task 9: Verify Data Ingestion
### Test the Pipeline

**1.** Generate a sample finding (if not done already):

- Go to GuardDuty → Settings → Sample findings

- Click Generate sample findings

**2.** Check SQS Queue:

- Go to SQS → ```gd-findings-queue```

- Click Send and receive messages → Poll for messages

- You should see messages appearing from GuardDuty findings

**3.** Wait 1-2 minutes for Splunk to poll SQS

**4.** Search in Splunk:

- Go to Search & Reporting

- Run:
```
index=main sourcetype=aws:guardduty:notifications
```

**5.** Verify fields are extracted:

- severity

- type

- accountId

- region

- resource.instanceDetails

## Task 10: Monitor and Validate

### Check for Errors
In Splunk, search for errors:
```
index=_internal source=*aws* ERROR
```
### Verify Message Flow
**1.** EventBridge: Check rule metrics in AWS Console

**2.** SNS: Check topic metrics (messages published)

**3.** SQS: Check queue metrics (messages received/deleted)

**4.** Splunk: Check input status in Add-on configuration

## Troubleshooting
### No data appearing in Splunk?

#### Check SQS Queue:

- Go to SQS console and poll for messages manually

- If messages appear, issue is with Splunk

- If no messages, issue is upstream (EventBridge or SNS)

#### Check EventBridge Rule:

- Ensure rule is Enabled

- Check CloudWatch metrics for the rule
- Verify event pattern matches GuardDuty findings

#### Check SNS Subscription:

- Ensure subscription status is Confirmed
- Check SNS topic metrics for published messages

#### Check Splunk Add-on Logs:
```
index=_internal source="*ta_aws*" ERROR
```

#### Verify IAM Permissions:

- Test Splunk user can access SQS using AWS CLI:
```
aws sqs receive-message --queue-url <QUEUE_URL>
```
#### Messages stuck in SQS?

- Check Splunk input is enabled

- Verify AWS credentials are correct

- Increase visibility timeout if Splunk processing is slow

#### EventBridge not triggering?

- Generate sample findings to test

- Check EventBridge rule event pattern syntax

- Verify GuardDuty is enabled and generating findings

## Best Practices

#### Performance
- Set appropriate SQS visibility timeout: 60-120 seconds for complex processing

- Monitor queue depth: Alert if messages are accumulating
- Use dead-letter queue: Capture failed messages for investigation

#### Security

- Use IAM roles instead of access keys when Splunk runs on AWS (EC2/ECS)

- Enable encryption at rest for SQS queue

- Rotate access keys regularly (every 90 days)

- Apply least privilege: Only grant necessary SQS permissions

#### Cost Optimization

- Set appropriate message retention: Default 4 days is usually sufficient

- Use long polling: Reduces empty API calls (Add-on handles this)

- Monitor SQS costs: Check AWS Cost Explorer

#### Monitoring

- Create Splunk alerts for high-severity GuardDuty findings

- Set up dashboards to visualize findings by severity, type, and resource

- Enable CloudWatch alarms for SQS queue depth

## Screenshots

### Pipeline Architecture
![Pipeline Architecture](images/Pipeline.png)  

### GuardDuty Findings Sample
![GuardDuty Findings Sample](images/GuardDutyFindingsSample.png)

### EventBridge Rule
![EventBridge Rule](images/EventBridgeRule.png)

### GuardDuty Findings Topic
![GuardDuty Findings Topic](images/GdFindingsTopic.png)

### GuardDuty Findings Queue
![GuardDuty Findings Queue](images/GdFindingsQueue.png)

### IAM User Activity in Splunk
![IAM User Activity](images/IAMUserSplunkGuard.png)

### Splunk Raw Logs
![Splunk Raw Logs](images/GuardSplunkRaw.png)

### GuardDuty Findings Splunk Dashboard
![GuardDuty Findings Splunk Dashboard](images/GuardDutyFindingsSplunkDash.png)


