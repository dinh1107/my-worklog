---
title: "Monitoring with Amazon CloudWatch"
date: 2026-09-14
weight: 11
chapter: false
pre: "<b>5.11. </b>"
---

# Monitoring Lambda with Amazon CloudWatch

After AWS IoT Rules invoke `SmartHomeIngest`, we need to verify whether the function runs successfully, how long each invocation takes, and whether errors occur while writing to DynamoDB or publishing to SNS.

Amazon CloudWatch provides two main monitoring components:

| Component | Purpose |
|---|---|
| CloudWatch Metrics | Tracks invocations, errors, duration, and throttling |
| CloudWatch Logs | Provides detailed logs for individual Lambda invocations |

In this section, we will inspect the Lambda metrics, open the `SmartHomeIngest` logs, and configure an appropriate log retention period.

## 5.11.1. Role of Amazon CloudWatch

CloudWatch makes it possible to observe the cloud processing pipeline without directly accessing the IoT device.

The current monitoring flow is:

```text
AWS IoT Rule
→ SmartHomeIngest
→ CloudWatch Metrics and Logs
```

CloudWatch helps answer the following questions:

- Was Lambda invoked by the AWS IoT Rule?
- Did the invocation complete successfully?
- Did an `AccessDeniedException` occur?
- Did the function exceed its timeout?
- How long did each invocation take?
- What payload did Lambda receive?

CloudWatch provides monitoring and operational logs. The primary application data remains stored in DynamoDB.

## 5.11.2. Resources to verify

Confirm the AWS Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

The following resources are used:

| Component | Value |
|---|---|
| Lambda function | `SmartHomeIngest` |
| Execution role | `SmartHomeIngestRole` |
| Log group | `/aws/lambda/SmartHomeIngest` |
| Recommended retention | `30 days` |

Lambda normally creates the log group automatically after its first invocation. Manual log group creation is unnecessary when the execution role has the required permissions.

## 5.11.3. Verify CloudWatch Logs permissions

Lambda requires the following permissions to create and write logs:

```text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
```

These permissions are included in the AWS managed policy:

```text
AWSLambdaBasicExecutionRole
```

Perform this check using the root or administrator account.

Navigate to:

```text
IAM
→ Roles
→ SmartHomeIngestRole
→ Permissions
```

Check the attached permission policies. If the following policy is already attached, no change is required:

```text
AWSLambdaBasicExecutionRole
```

If it is missing, choose:

```text
Add permissions
→ Attach policies
```

Search for and select:

```text
AWSLambdaBasicExecutionRole
```

Then choose:

```text
Add permissions
```

This policy only provides basic log-writing permissions. It does not grant access to DynamoDB or SNS.

After completing the check, sign out of the root account and continue as the `dinh-fcj` IAM user.

## 5.11.4. View Lambda metrics

Navigate to:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Monitor
```

Under **Metrics**, select an appropriate time range, such as:

```text
Last 1 hour
```

or:

```text
Last 3 hours
```

Inspect the following metrics:

| Metric | Meaning | Expected result |
|---|---|---|
| Invocations | Total number of function invocations | Increases when telemetry or events arrive |
| Errors | Number of failed invocations | `0` |
| Duration | Function processing time | Lower than the configured timeout |
| Throttles | Invocations rejected by concurrency limits | `0` |
| Concurrent executions | Simultaneous function executions | Low for this workshop |

Telemetry is sent approximately every 30 seconds. Therefore, **Invocations** should increase steadily while the system is active.

Events such as `gas_alarm_started` and `door_brute_force_detected` create additional invocations.

CloudWatch metrics can take several minutes to update. A graph that does not update immediately does not necessarily indicate a failure.

![SmartHomeIngest metrics](/images/5.11.4.png)


## 5.11.5. Open CloudWatch Logs

From the Lambda **Monitor** tab, choose:

```text
View CloudWatch logs
```

Alternatively, navigate directly to:

```text
Amazon CloudWatch
→ Logs
→ Log groups
→ /aws/lambda/SmartHomeIngest
```

The log group contains a list of **Log streams**. The latest stream normally appears first when the list is sorted by **Last event time**.

Open the latest log stream.

A successful Lambda invocation normally contains:

```text
START RequestId: ...
...
END RequestId: ...
REPORT RequestId: ...
```

The entries have the following meanings:

| Log entry | Meaning |
|---|---|
| `START` | Lambda started processing the request |
| Custom logs | Data written using `print()` or a logger |
| `END` | Lambda finished processing the request |
| `REPORT` | Duration, memory usage, and billing information |

Example report:

```text
REPORT RequestId: ...
Duration: 120.00 ms
Billed Duration: 121 ms
Memory Size: 128 MB
Max Memory Used: 75 MB
```

If the source code records processing information, verify:

- Whether `recordType` is `telemetry` or `event`.
- The message `messageId`.
- The DynamoDB write result.
- The SNS publish result.
- Whether a duplicate message was ignored.

![SmartHomeIngest log stream](/images/5.11.5.png)





## 5.11.6. Search for errors

Use the search field in the log stream to find:

```text
ERROR
Exception
AccessDenied
Task timed out
```

Common errors include:

| Error | Possible cause |
|---|---|
| `AccessDeniedException` | The execution role lacks DynamoDB or SNS permissions |
| `ResourceNotFoundException` | Incorrect table, topic, or Region |
| `ConditionalCheckFailedException` | The same `messageId` has already been stored |
| `Task timed out` | Processing exceeded the Lambda timeout |
| `KeyError` | A required payload property is missing |
| `JSONDecodeError` | The payload is not valid JSON |

In this project, a `ConditionalCheckFailedException` can occur when a message with the same `messageId` is sent again. If duplicate handling is already implemented, this does not represent a serious system failure.

## 5.11.7. Configure log retention

CloudWatch Logs can store log data indefinitely by default. For this workshop, set the retention period to 30 days to control storage usage and cost.

Navigate to:

```text
Amazon CloudWatch
→ Logs
→ Log groups
```

Find:

```text
/aws/lambda/SmartHomeIngest
```

In the **Retention** column, choose the current value and set:

```text
30 days
```

Choose:

```text
Save
```

This configuration does not stop Lambda from writing new logs. CloudWatch only removes log events older than the selected retention period.

A separate screenshot is not required for this step.

## 5.11.8. Metrics compared with Logs

| Requirement | Check |
|---|---|
| Verify whether Lambda was invoked | Invocations |
| Count failed invocations | Errors |
| Check processing time | Duration |
| Determine the cause of an error | CloudWatch Logs |
| Inspect the received payload | Custom log entries |
| Identify a permission issue | Search for `AccessDeniedException` |
| Identify a timeout | Search for `Task timed out` |

Metrics provide an overview of the function, while Logs provide detailed information about individual invocations.

## 5.11.9. Quick verification

When a new telemetry message arrives:

1. The AWS IoT Rule invokes `SmartHomeIngest`.
2. The **Invocations** metric increases.
3. The **Errors** metric remains at `0`.
4. A new log event appears in a log stream.
5. The invocation ends with `END` and `REPORT`.
6. A new item is written to DynamoDB.

When an alert event occurs:

1. Lambda receives `recordType` set to `event`.
2. The event is stored in DynamoDB.
3. Lambda publishes to SNS.
4. CloudWatch records the processing result.
5. SNS sends an alert email to the confirmed subscription.

The complete end-to-end test will be presented in section 5.12.

## 5.11.10. Common issues

### The log group cannot be found

Verify that:

- Lambda has been invoked at least once.
- The selected Region is `ap-southeast-1`.
- `SmartHomeIngestRole` has `AWSLambdaBasicExecutionRole`.
- Five to ten minutes have passed since the invocation.

### The log group exists but has no recent stream

Verify that:

- Both AWS IoT Rules are `Enabled`.
- The rule action references `SmartHomeIngest`.
- The MQTT topic matches the rule SQL statement.
- The selected CloudWatch time range is correct.

### The IAM user cannot open CloudWatch Logs

If an `AccessDenied` error appears, the administrator must check the CloudWatch read permissions assigned to `dinh-fcj` or `SmartHome-Developers`.

These are separate permissions:

- `SmartHomeIngestRole` allows Lambda to write logs.
- The IAM policy of `dinh-fcj` allows the user to open and read logs.

### Invocations and Errors both increase

Open the latest log stream and search for:

```text
ERROR
Exception
AccessDenied
```

Use the detailed log message to identify the actual cause instead of relying only on the graph.

## 5.11.11. Result

After completing this section:

- `SmartHomeIngest` can write to CloudWatch Logs.
- Invocations, Errors, and Duration are monitored.
- `/aws/lambda/SmartHomeIngest` contains log data.
- Individual Lambda invocations can be inspected.
- Log retention is configured for 30 days.
- CloudWatch can identify Lambda, DynamoDB, and SNS errors.

## References

- [Sending Lambda logs to CloudWatch Logs](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html)
- [Using CloudWatch metrics with Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics.html)
- [Working with CloudWatch log groups and streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)