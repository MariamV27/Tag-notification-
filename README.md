# Notification System for Untagged EC2 Instances

This project aims to send email notifications for EC2 instances that lack a specific tag (e.g., a tag named `Environment` without an assigned value). The solution leverages AWS services to automate the detection and notification process.

[](arquitectura.png)

## Solution Overview

1. **CloudWatch Events (EventBridge):** Triggers periodically to check for EC2 instances.
2. **AWS Lambda:** Executes a function that identifies instances missing the required tag or with a missing value.
3. **Amazon SNS:** Sends email notifications whenever untagged or misconfigured instances are detected.

With this setup, you can ensure better governance and compliance of your AWS resources.

### Steps to Set Up Notifications for Untagged EC2 Instances

### Create an SNS Topic for Email Notifications

1. Go to the **Amazon SNS Console** and create a topic (e.g., `MissingTagNotifications`).
2. Choose the topic type as **Standard**.
3. Give it a name and create the topic.
4. Create a subscription to this SNS topic with your email address.
5. Confirm the subscription via the email you receive.

### Create a Lambda Function to Check EC2 Instances

1. Go to the **AWS Lambda Console** and create a new Lambda function.
2. Choose **"Author from scratch"**, give it a name (e.g., `CheckEC2Tags`), and use a **Python runtime**.
3. Attach the necessary permissions to the Lambda function:
   - **EC2 DescribeInstances** permission to read instance tags.
   - **SNS Publish** permission to send notifications.

#### Lambda Function

import boto3
import os

ec2 = boto3.client('ec2')
sns = boto3.client('sns')

def lambda_handler(event, context):
    # Replace with your SNS topic ARN
    sns_topic_arn = os.environ['SNS_TOPIC_ARN']
    
    # Describe EC2 instances
    instances = ec2.describe_instances()
    
    missing_tag_instances = []
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            # Get tags
            tags = instance.get('Tags', [])
            # Check if the 'Environment' tag is missing or empty
            if not any(tag['Key'] == 'Environment' and tag['Value'] for tag in tags):
                missing_tag_instances.append(instance['InstanceId'])
    
    # If any instances are missing the 'Environment' tag, send an SNS notification
    if missing_tag_instances:
        message = f"These EC2 instances are missing the 'Environment' tag: {missing_tag_instances}"
        sns.publish(
            TopicArn=sns_topic_arn,
            Subject="EC2 Instances Missing 'Environment' Tag",
            Message=message
        )
    
    return {
        'statusCode': 200,
        'body': 'Notification sent for missing tag instances.'
    }
## Configure Lambda Environment Variables

After creating the Lambda function, set the following environment variable:

- **SNS_TOPIC_ARN**: Set this to the ARN of the SNS topic created earlier.

## Create a CloudWatch Rule (EventBridge)

1. Go to the **CloudWatch Console** and create a rule.
2. Select **Event Source** as **EventBridge** and choose a schedule (e.g., every five minutes).
3. To run the EventBridge rule every 5 minutes, use the following cron expression:

   ```cron
   cron(0/5 * * * ? *)
