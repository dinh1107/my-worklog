---
title: "Configuring Amazon SNS Alerts"
date: 2026-09-14
weight: 10
chapter: false
pre: "<b>5.10. </b>"
---

# Configuring Amazon SNS Email Alerts

In the previous section, AWS IoT Rules were configured to forward telemetry and events to the `SmartHomeIngest` Lambda function.

Telemetry only needs to be stored periodically. Safety events, however, must notify the user as soon as they occur. In this section, Amazon Simple Notification Service — Amazon SNS — is used to deliver email alerts.

The system supports three notification types:

| Event type | Meaning |
|---|---|
| `gas_alarm_started` | The gas level exceeds the configured threshold |
| `gas_alarm_cleared` | The gas level has returned to a safe state |
| `door_brute_force_detected` | An incorrect door password was entered three times |

## 5.10.1. Role of Amazon SNS

Amazon SNS is a publish/subscribe notification service.

In this workshop:

- `SmartHomeIngest` is the message publisher.
- `SmartHomeAlerts` is the distribution channel.
- The registered email address is the notification endpoint.

The alert flow is:

```text
MQTT event
→ AWS IoT Rule
→ SmartHomeIngest
→ Amazon SNS
→ User email
```

SNS separates event-processing logic from the notification delivery method. Additional recipients or endpoints can therefore be added later without changing the IoT device.

## 5.10.2. Configuration overview

Verify the AWS Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Use the following configuration:

| Component | Value |
|---|---|
| Topic type | `Standard` |
| Topic name | `SmartHomeAlerts` |
| Display name | `SmartHome Alerts` |
| Subscription protocol | `Email` |
| Lambda function | `SmartHomeIngest` |
| Lambda execution role | `SmartHomeIngestRole` |
| Environment variable | `SNS_TOPIC_ARN` |

A **Standard** topic is suitable because the system requires prompt notification and does not require strict ordering between email messages.

## 5.10.3. Create the SNS topic

Perform this step with the root or administrator account if the IAM user does not have permission to create SNS topics.

### Step 1: Open Amazon SNS

In the AWS Management Console, search for:

```text
Amazon Simple Notification Service
```

Navigate to:

```text
Amazon SNS
→ Topics
→ Create topic
```

### Step 2: Enter the topic information

Under **Details**, enter:

| Property | Value |
|---|---|
| Type | `Standard` |
| Name | `SmartHomeAlerts` |
| Display name | `SmartHome Alerts` |

Keep the remaining settings at their default values:

- FIFO is not required.
- A custom email retry policy is not required.
- A customer-managed KMS key is not required.
- Keep the default access policy.

Choose:

```text
Create topic
```

After the topic is created, AWS displays an ARN in the following format:

```text
arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts
```

Copy this ARN for the Lambda configuration.

![SmartHomeAlerts topic details](/images/5.10.3.png)

> Evidence: capture the topic name, Standard type, and ARN. Mask part of the AWS account ID before publishing the image.

## 5.10.4. Create an email subscription

On the `SmartHomeAlerts` topic details page, choose:

```text
Create subscription
```

Enter:

| Property | Value |
|---|---|
| Topic ARN | ARN of `SmartHomeAlerts` |
| Protocol | `Email` |
| Endpoint | Email address that receives the alerts |

Choose:

```text
Create subscription
```

The initial subscription state will be:

```text
Pending confirmation
```

### Confirm the email address

Open the registered mailbox and locate an email with a subject similar to:

```text
AWS Notification - Subscription Confirmation
```

Open the email and choose:

```text
Confirm subscription
```

Return to Amazon SNS and refresh the page. The subscription state must change to:

```text
Confirmed
```

If the email does not appear:

- Check the Spam or Junk folder.
- Verify the subscription email address.
- Wait a few minutes and resend the confirmation if necessary.
- Avoid creating multiple subscriptions for the same address.

![Confirmed email subscription](/images/5.10.4.png)



## 5.10.5. Grant SNS publish permission to Lambda

The Lambda execution role requires `sns:Publish` permission before the function can send notifications.

Perform this step with the root or administrator account.

Navigate to:

```text
IAM
→ Roles
→ SmartHomeIngestRole
→ Permissions
→ Add permissions
→ Create inline policy
→ JSON
```

Enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishSmartHomeAlerts",
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts"
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with the actual AWS account ID.

Choose **Next** and enter the policy name:

```text
SmartHomeIngestSNSPublish
```

Then choose:

```text
Create policy
```

This policy allows the function to publish only to `SmartHomeAlerts` instead of granting access to every SNS topic in the account. This follows the principle of least privilege.

## 5.10.6. Configure the SNS topic in Lambda

Sign out of the root or administrator account and sign in again as the IAM user `dinh-fcj`.

Navigate to:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Configuration
→ Environment variables
→ Edit
```

Add:

| Key | Value |
|---|---|
| `SNS_TOPIC_ARN` | ARN of the `SmartHomeAlerts` topic |

Example:

```text
SNS_TOPIC_ARN
arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts
```

Choose:

```text
Save
```

The environment variable name must exactly match the name used by the `SmartHomeIngest` source code.

Using an environment variable avoids hard-coding the topic ARN. If the topic changes, only the Lambda configuration needs to be updated.

## 5.10.7. Lambda notification logic

The `SmartHomeIngest` function creates an SNS client using the AWS SDK for Python:

```python
import boto3
import os

sns = boto3.client("sns")
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]
```

When an alert event is received, the function publishes a message:

```python
sns.publish(
    TopicArn=SNS_TOPIC_ARN,
    Subject=subject,
    Message=message
)
```

Notifications are generated only for records containing:

```json
{
  "recordType": "event"
}
```

Example event mappings:

| Event type | Email subject |
|---|---|
| `gas_alarm_started` | `ALERT: Gas leak detected` |
| `gas_alarm_cleared` | `NOTICE: Gas level is safe` |
| `door_brute_force_detected` | `ALERT: Repeated incorrect door password` |

Telemetry messages sent every 30 seconds do not generate email notifications. This prevents unnecessary messages.

## 5.10.8. Test the SNS topic independently

Before testing the complete system, verify that the SNS topic and subscription work independently.

Navigate to:

```text
Amazon SNS
→ Topics
→ SmartHomeAlerts
→ Publish message
```

Enter:

| Property | Value |
|---|---|
| Subject | `Smart Home SNS Test` |
| Message body | `Amazon SNS email notification is working.` |

Choose:

```text
Publish message
```

Check the registered mailbox. Receiving the test email confirms that the topic and subscription are working.

## 5.10.9. Verify an actual alert

When one of the three supported events occurs, SNS sends an email to the confirmed address.

Example gas event:

```json
{
  "eventType": "gas_alarm_started",
  "gasRaw": 2000,
  "threshold": 1500
}
```

Expected result:

- `SmartHomeIngest` is invoked.
- The event is stored in DynamoDB.
- SNS sends an alert email.
- The email contains the device, event type, and event time.

![Smart home alert email](/images/5.10.9.png)



## 5.10.10. Common issues

### The subscription remains Pending confirmation

Verify that:

- The confirmation email is not in the Spam or Junk folder.
- The subscription endpoint contains the correct email address.
- The **Confirm subscription** link has been opened.
- The SNS console was refreshed after confirmation.

SNS does not deliver notifications to an unconfirmed email endpoint.

### Lambda returns AccessDeniedException

If CloudWatch shows an `sns:Publish` error, verify that:

- The execution role is `SmartHomeIngestRole`.
- The role contains `SmartHomeIngestSnsPublishPolicy`.
- The policy resource matches the `SmartHomeAlerts` ARN.
- Lambda and the SNS topic are in the same Region.

### Lambda succeeds, but no email is received

Verify that:

- The subscription state is `Confirmed`.
- `SNS_TOPIC_ARN` contains the correct topic ARN.
- The payload has `recordType` set to `event`.
- The `eventType` is supported by the Lambda function.
- The message is not in the Spam or Promotions folder.

### Telemetry does not generate email

This is expected. Telemetry is stored periodically and is not treated as an alert event.

## 5.10.11. Result

After completing this section:

- The `SmartHomeAlerts` SNS topic has been created.
- The email subscription has been confirmed.
- `SmartHomeIngestRole` has `sns:Publish` permission.
- Lambda receives the topic ARN through `SNS_TOPIC_ARN`.
- Gas and security events can generate email alerts.
- Periodic telemetry does not create unnecessary notifications.

## References

- [Creating an Amazon SNS topic](https://docs.aws.amazon.com/sns/latest/dg/sns-create-topic.html)
- [Amazon SNS email subscription setup](https://docs.aws.amazon.com/sns/latest/dg/sns-email-notifications.html)
- [AWS Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)
- [AWS Lambda execution roles](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)