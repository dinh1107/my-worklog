---
title: "End-to-End Data Flow Testing"
date: 2026-09-14
weight: 12
chapter: false
pre: "<b>5.12. </b>"
---

# End-to-End Data Flow Testing

After completing AWS IoT Core, IoT Rules, Lambda, DynamoDB, SNS, and CloudWatch, the complete cloud data flow must be verified.

The purpose of this section is to confirm that an MQTT message can travel through the entire architecture:

```text
MQTT message
→ AWS IoT Core
→ AWS IoT Rule
→ SmartHomeIngest
→ DynamoDB
→ CloudWatch
→ SNS for alert events
```

The tests use the AWS resources created in the previous sections. No hardware or Home Assistant configuration is required.

## 5.12.1. Test objectives

The following results must be verified:

- AWS IoT Core receives telemetry and event messages.
- Each AWS IoT Rule is triggered by the correct topic.
- `SmartHomeIngest` processes the payload successfully.
- Telemetry and events are stored in DynamoDB.
- Alert events are published to Amazon SNS.
- CloudWatch records each Lambda invocation.
- A repeated `messageId` does not create another item.
- The system has no permission or timeout errors.

## 5.12.2. Verify the resources

Confirm the AWS Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Verify the following resources:

| Component | Expected value |
|---|---|
| IoT Thing | `ha-gateway-home01` |
| Telemetry rule | `SmartHomeTelemetryRule` — Enabled |
| Events rule | `SmartHomeEventsRule` — Enabled |
| Lambda function | `SmartHomeIngest` |
| DynamoDB table | `SmartHomeTelemetry` — Active |
| SNS topic | `SmartHomeAlerts` |
| Email subscription | Confirmed |
| CloudWatch log group | `/aws/lambda/SmartHomeIngest` |

Resolve any inactive resource before testing to avoid misleading results.

## 5.12.3. Monitor MQTT messages

Navigate to:

```text
AWS IoT Core
→ Test
→ MQTT test client
```

Open:

```text
Subscribe to a topic
```

Enter the topic filter:

```text
smarthome/home01/esp32-01/#
```

Choose:

```text
Subscribe
```

The `#` character is a multi-level wildcard. This filter displays messages from both:

```text
smarthome/home01/esp32-01/telemetry
smarthome/home01/esp32-01/events
```

While the system is running, telemetry should appear approximately every 30 seconds.

Verify that the payload contains the main data properties, such as:

- `messageId`
- Message timestamp
- `homeId`
- `deviceId`
- Temperature
- Humidity
- Raw gas ADC value
- Door, light, or fan state

The exact property names must match the payload structure used by `SmartHomeIngest`.

![Messages in MQTT Test Client](/images/5.12.3.png)



## 5.12.4. Test telemetry processing

If telemetry is already being published automatically, select a recent message and record its `messageId`.

For a controlled test, open:

```text
Publish to a topic
```

Enter:

```text
smarthome/home01/esp32-01/telemetry
```

Use a payload matching the implemented schema:

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-telemetry-20260914-001",
  "timestamp": "2026-09-14T20:00:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "temperature": 29.5,
  "humidity": 72,
  "gasRaw": 1020,
  "doorState": "CLOSED",
  "lightState": "OFF",
  "fanState": "OFF",
  "source": "workshop-test"
}
```

Use a new `messageId` and timestamp for each independent test. Otherwise, Lambda can identify the message as a duplicate.

Choose:

```text
Publish
```

### Expected result

1. `SmartHomeTelemetryRule` receives the message.
2. The rule adds:

```json
{
  "recordType": "telemetry"
}
```

3. The rule invokes `SmartHomeIngest`.
4. Lambda normalizes the data.
5. A new item is written to `SmartHomeTelemetry`.
6. CloudWatch records no errors.
7. SNS does not send an email because telemetry is not an alert event.

## 5.12.5. Verify telemetry in DynamoDB

Navigate to:

```text
Amazon DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
```

Use the search or filter feature to find:

```text
messageId = test-telemetry-20260914-001
```

If the console cannot filter directly by `messageId`, inspect the latest items or sort them by `recordKey`.

The telemetry item should contain:

| Property | Expected result |
|---|---|
| `deviceKey` | Identifies `home01` and `esp32-01` |
| `recordKey` | Contains the record type and time |
| `recordType` | `telemetry` |
| `messageId` | Matches the published message |
| `temperature` | `29.5` in the test payload |
| `humidity` | `72` in the test payload |
| `gasRaw` | `1020` in the test payload |
| `expiresAt` | Unix epoch time in seconds for TTL |

The exact formats of `deviceKey` and `recordKey` depend on the `SmartHomeIngest` implementation. Both key attributes must be present.

## 5.12.6. Test a gas alarm event

In the MQTT Test Client, open:

```text
Publish to a topic
```

Enter:

```text
smarthome/home01/esp32-01/events
```

Use:

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-gas-started-20260914-001",
  "timestamp": "2026-09-14T20:05:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "eventType": "gas_alarm_started",
  "severity": "critical",
  "gasRaw": 2000,
  "threshold": 1500,
  "source": "workshop-test"
}
```

Choose:

```text
Publish
```

This is a simulated cloud test message. The `gasRaw` property represents a raw ADC value, not a calibrated ppm measurement.

### Expected result

1. `SmartHomeEventsRule` receives the message.
2. The rule adds `recordType` with the value `event`.
3. `SmartHomeIngest` is invoked.
4. The event is stored in DynamoDB.
5. Lambda publishes a notification to `SmartHomeAlerts`.
6. SNS sends an email to the confirmed subscription.
7. CloudWatch records a successful invocation.

The alert email screenshot was already included in section 5.10 and does not need to be repeated.

## 5.12.7. Test the gas-cleared event

Publish another message to the events topic:

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-gas-cleared-20260914-001",
  "timestamp": "2026-09-14T20:06:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "eventType": "gas_alarm_cleared",
  "severity": "info",
  "gasRaw": 1000,
  "threshold": 1500,
  "source": "workshop-test"
}
```

Expected result:

- The event is stored in DynamoDB.
- `recordType` is `event`.
- SNS sends an email indicating that the gas level is safe.
- CloudWatch records no error.

This event informs the user that the hazardous state has ended.

## 5.12.8. Test the repeated incorrect password event

Publish the following message to:

```text
smarthome/home01/esp32-01/events
```

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-door-failed-20260914-001",
  "timestamp": "2026-09-14T20:07:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "eventType": "door_brute_force_detected",
  "severity": "critical",
  "failedAttempts": 3,
  "source": "workshop-test"
}
```

Expected result:

- `SmartHomeEventsRule` invokes Lambda.
- The event is stored in DynamoDB.
- SNS sends a security alert email.
- The email does not contain the door password.
- CloudWatch records a successful invocation.

Only the number of failed attempts is sent. Passwords and entered keys must never be included in the cloud payload.

## 5.12.9. Verify event records

Return to:

```text
Amazon DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
```

Verify that recent items contain:

```text
recordType = event
```

The following event types should be present:

```text
gas_alarm_started
gas_alarm_cleared
door_brute_force_detected
```

![Telemetry and events in DynamoDB](/images/5.12.9.png)



## 5.12.10. Test duplicate protection

The duplicate-protection mechanism uses `messageId` to avoid storing the same message more than once.

Perform the following test:

1. Select the telemetry payload used earlier.
2. Keep the same `messageId`:

```text
test-telemetry-20260914-001
```

3. Publish the payload again to the telemetry topic.
4. Inspect DynamoDB.

Expected result:

- The AWS IoT Rule can invoke Lambda again.
- DynamoDB does not create a second item for the same message.
- Lambda logs that the message already exists or was skipped.
- The duplicate does not cause an uncontrolled invocation failure.

This protection is important because MQTT QoS 1 can deliver a message more than once.

A separate screenshot is not required. The CloudWatch evidence in the next step can include the duplicate-handling log.

## 5.12.11. Verify CloudWatch Logs

Navigate to:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Monitor
→ View CloudWatch logs
```

Open the latest stream in:

```text
/aws/lambda/SmartHomeIngest
```

Verify that:

- `START RequestId` is present.
- Lambda receives the expected `recordType`.
- There is no `AccessDeniedException`.
- There is no `ResourceNotFoundException`.
- There is no `Task timed out`.
- `END RequestId` is present.
- `REPORT RequestId` is present.

For a duplicate message, the log should indicate that the message was skipped or already existed. Duplicate detection should not cause the entire invocation to fail.




## 5.12.12. Test result summary

| Scenario | IoT Rule | DynamoDB | SNS email | CloudWatch |
|---|---|---|---|---|
| Telemetry | Successful | New item | Not sent | No error |
| Gas alarm started | Successful | Event stored | Received | No error |
| Gas level cleared | Successful | Event stored | Received | No error |
| Three failed door attempts | Successful | Event stored | Received | No error |
| Repeated `messageId` | Lambda invoked | No duplicate item | Not resent if skipped | Duplicate logged |

## 5.12.13. Completion checklist

The test is complete when:

- [x] MQTT Test Client receives telemetry.
- [x] MQTT Test Client receives events.
- [x] `SmartHomeTelemetryRule` operates correctly.
- [x] `SmartHomeEventsRule` operates correctly.
- [x] Lambda has no failed invocation.
- [x] DynamoDB stores telemetry and events.
- [x] SNS sends alert emails.
- [x] CloudWatch contains invocation logs.
- [x] A duplicate message does not create another item.
- [x] No passwords or secrets are stored in cloud data.

## 5.12.14. Common issues

### No messages appear in MQTT Test Client

Verify that:

- The Region is `ap-southeast-1`.
- The topic filter is `smarthome/home01/esp32-01/#`.
- MQTT topic names use the correct character casing.
- The existing source system is publishing data.

### MQTT receives data, but Lambda is not invoked

Verify that:

- The IoT Rule is `Enabled`.
- The rule SQL statement uses the correct topic.
- The rule action references `SmartHomeIngest`.
- The Lambda resource policy allows `iot.amazonaws.com`.

### Lambda runs, but DynamoDB has no item

Search CloudWatch Logs for:

```text
AccessDeniedException
ResourceNotFoundException
KeyError
```

Also verify the table name and required payload properties.

### DynamoDB receives the event, but no email arrives

Verify that:

- The SNS subscription is `Confirmed`.
- `SNS_TOPIC_ARN` contains the correct ARN.
- `SmartHomeIngestRole` has `sns:Publish`.
- The email is not in the Spam or Junk folder.
- The Lambda function supports the published `eventType`.

## 5.12.15. Result

After completing this section, the complete cloud data flow has been verified:

- AWS IoT Core receives MQTT messages.
- IoT Rules classify telemetry and events.
- Lambda processes and normalizes the data.
- DynamoDB stores time-series records.
- SNS sends email for alert events.
- CloudWatch provides metrics and execution logs.
- `messageId` prevents duplicate records.

The system now satisfies the requirements for cloud ingestion, processing, storage, notification, and monitoring of smart home data.

## References

- [View MQTT messages with the AWS IoT MQTT client](https://docs.aws.amazon.com/iot/latest/developerguide/view-mqtt-messages.html)
- [MQTT support in AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
- [Getting started with Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStartedDynamoDB.html)
- [Viewing CloudWatch logs for Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-view.html)