---
title: "Creating AWS IoT Rules"
date: 2026-09-14
weight: 9
chapter: false
pre: "<b>5.9. </b>"
---

# Creating AWS IoT Rules to Invoke Lambda

In the previous section, we created the `SmartHomeIngest` Lambda function to process smart home data. However, the function does not automatically receive MQTT messages from AWS IoT Core.

In this section, we will create two AWS IoT Rules:

- `SmartHomeTelemetryRule` for periodic device telemetry.
- `SmartHomeEventsRule` for events that require immediate processing.

Both rules forward their payloads to the `SmartHomeIngest` Lambda function.

## 5.9.1. Role of the AWS IoT Rules Engine

The AWS IoT Rules Engine evaluates messages published to MQTT topics. When a topic matches a rule's SQL statement, AWS IoT Core performs the configured action.

The action in this workshop invokes a Lambda function:

```text
MQTT topic → AWS IoT Rule → SmartHomeIngest → DynamoDB
```

AWS IoT policies and AWS IoT Rules serve different purposes:

| Component | Purpose |
|---|---|
| AWS IoT Policy | Allows a device to connect, publish, or subscribe |
| AWS IoT Rule | Selects and routes MQTT messages to AWS services |

We use separate rules so the Lambda function can clearly distinguish telemetry records from event records.

## 5.9.2. Required configuration

Verify the AWS Region before continuing:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

The following resources are used:

| Component | Value |
|---|---|
| Lambda function | `SmartHomeIngest` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| IoT SQL version | `2016-03-23` |
| Telemetry rule | `SmartHomeTelemetryRule` |
| Events rule | `SmartHomeEventsRule` |

Sign in with the IAM user `dinh-fcj`. If an `AccessDenied` error appears, use the root or administrator account to verify the permissions attached to the `SmartHome-Developers` group.

## 5.9.3. Create the telemetry rule

### Step 1: Open AWS IoT Rules

In the AWS Management Console, navigate to:

```text
AWS IoT Core
→ Message routing
→ Rules
```

Choose:

```text
Create rule
```

### Step 2: Enter the rule information

Enter the following values:

| Property | Value |
|---|---|
| Rule name | `SmartHomeTelemetryRule` |
| Description | `Forward smart home telemetry messages to SmartHomeIngest Lambda` |

Choose **Next**.

### Step 3: Enter the SQL statement

Select the following SQL version:

```text
2016-03-23
```

Enter:

```sql
SELECT *, 'telemetry' AS recordType
FROM 'smarthome/home01/esp32-01/telemetry'
```

The statement contains the following components:

| Component | Meaning |
|---|---|
| `SELECT *` | Preserves all properties from the incoming payload |
| `'telemetry' AS recordType` | Adds a `recordType` property with the value `telemetry` |
| `FROM '...'` | Selects messages published to the telemetry topic |

The `recordType` property allows `SmartHomeIngest` to select the correct processing logic without having to infer the message type from the payload.

Choose **Next**.

### Step 4: Configure the Lambda action

Under **Rule actions**, select:

```text
Action 1
→ Lambda
```

Select the following function:

```text
SmartHomeIngest
```

If the console displays a version or alias option, leave it at the default value to invoke the current function version.

Choose **Next**, review the configuration, and then choose:

```text
Create
```

When a Lambda action is configured through the AWS Console, the console normally adds the permission required for AWS IoT Core to invoke the function. Approve the permission request if it is displayed.

![SmartHomeTelemetryRule configuration](/images/5.9.3.png)

> Evidence: capture the rule details showing the rule name, SQL statement, and the `SmartHomeIngest` Lambda action.

## 5.9.4. Create the events rule

Return to:

```text
AWS IoT Core
→ Message routing
→ Rules
→ Create rule
```

### Step 1: Enter the rule information

| Property | Value |
|---|---|
| Rule name | `SmartHomeEventsRule` |
| Description | `Forward smart home event messages to SmartHomeIngest Lambda` |

Choose **Next**.

### Step 2: Enter the SQL statement

Select:

```text
2016-03-23
```

Enter:

```sql
SELECT *, 'event' AS recordType
FROM 'smarthome/home01/esp32-01/events'
```

This rule is activated only when a message is published to the events topic. The payload sent to Lambda is supplemented with:

```json
{
  "recordType": "event"
}
```

All other properties from the original payload are preserved.

### Step 3: Configure the Lambda action

Select:

```text
Action 1
→ Lambda
→ SmartHomeIngest
```

Choose **Next**, review the configuration, and choose **Create**.

![SmartHomeEventsRule configuration](/images/5.9.4.png)

> Evidence: capture the rule details showing the rule name, SQL statement, and Lambda action.

## 5.9.5. Verify both rules

Navigate to:

```text
AWS IoT Core
→ Message routing
→ Rules
```

The rule list must contain:

| Rule | State | Lambda action |
|---|---|---|
| `SmartHomeTelemetryRule` | Enabled | `SmartHomeIngest` |
| `SmartHomeEventsRule` | Enabled | `SmartHomeIngest` |

If either rule is **Disabled**, enable it before testing the data flow.

![Both AWS IoT Rules enabled](/images/5.9.5.png)


## 5.9.6. Verify the Lambda invocation permissions

Configuring the rule action is not sufficient if AWS IoT Core does not have permission to invoke the Lambda function.

Navigate to:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Configuration
→ Permissions
→ Resource-based policy statements
```

Verify that the function contains permission statements for AWS IoT Core:

| Property | Expected value |
|---|---|
| Principal | `iot.amazonaws.com` |
| Action | `lambda:InvokeFunction` |
| Source rule | `SmartHomeTelemetryRule` or `SmartHomeEventsRule` |

The corresponding source ARNs have the following format:

```text
arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:rule/SmartHomeTelemetryRule
```

and:

```text
arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:rule/SmartHomeEventsRule
```

Replace `<ACCOUNT_ID>` with your AWS account ID. Mask part of the account ID before publishing the screenshot on a public website.

![AWS IoT permissions for SmartHomeIngest](/images/5.9.6.png)


### Why is this permission not part of the execution role?

The two permission types have different purposes:

- `SmartHomeIngestRole` allows the Lambda function to access DynamoDB, SNS, and CloudWatch.
- The Lambda resource-based policy allows AWS IoT Core to invoke `SmartHomeIngest`.

If a required permission statement is missing, edit the IoT Rule, reselect `SmartHomeIngest` as the Lambda action, and save the rule. The AWS Console will request or add the required permission.

Avoid granting unrestricted invocation permission using a wildcard source when it is not required.

## 5.9.7. Perform a quick data-flow check

Because the existing system is already sending data to AWS IoT Core, there is no need to configure the device or Home Assistant again.

After enabling both rules:

1. Wait for a new telemetry message.
2. Open the `SmartHomeIngest` Lambda function.
3. Check the **Invocations** metric in the **Monitor** tab.
4. Open the `SmartHomeTelemetry` DynamoDB table.
5. Verify that a new item has `recordType` set to `telemetry`.
6. When an event occurs, verify that its item has `recordType` set to `event`.

The complete test using MQTT Test Client, DynamoDB, SNS, and CloudWatch will be presented in section 5.12.

## 5.9.8. Common issues

### The rule is not triggered

Verify that:

- The rule state is `Enabled`.
- The selected Region is `ap-southeast-1`.
- The topic in the SQL statement exactly matches the actual MQTT topic.
- The topic does not contain incorrect spaces or character casing.

### Lambda does not receive new invocations

Verify that:

- The rule has the `SmartHomeIngest` Lambda action.
- The resource-based policy contains the `iot.amazonaws.com` principal.
- The source ARN references the correct rule.
- The Lambda function and IoT Rule are in the same Region.

### Lambda is invoked, but DynamoDB receives no item

Open the CloudWatch Logs for `SmartHomeIngest` and verify that:

- The payload uses valid JSON.
- `recordType` is either `telemetry` or `event`.
- The Lambda execution role can write to `SmartHomeTelemetry`.
- There are no `AccessDeniedException` or missing-field errors.

## 5.9.9. Result

After completing this section:

- The telemetry topic is connected to `SmartHomeTelemetryRule`.
- The events topic is connected to `SmartHomeEventsRule`.
- Both rules invoke `SmartHomeIngest`.
- Each payload receives a `recordType` property.
- AWS IoT Core has permission to invoke the Lambda function.
- MQTT data can continue through the AWS processing and storage pipeline.

## References

- [AWS IoT Rules](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html)
- [AWS IoT SQL reference](https://docs.aws.amazon.com/iot/latest/developerguide/iot-sql-reference.html)
- [AWS IoT Lambda rule action](https://docs.aws.amazon.com/iot/latest/developerguide/lambda-rule-action.html)