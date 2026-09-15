---
title: "Building the SmartHomeIngest Lambda Function"
date: 2026-09-14
weight: 8
chapter: false
pre: "<b>5.8. </b>"
---

# Building the SmartHomeIngest Lambda Function

In this section, we will create the `SmartHomeIngest` Lambda function to process telemetry and events received from AWS IoT Core.

The function performs the following tasks:

- Validates incoming data.
- Distinguishes telemetry from events.
- Generates DynamoDB keys.
- Converts floating-point values for DynamoDB.
- Generates a TTL expiration timestamp.
- Prevents duplicate records.
- Publishes important alerts to SNS.
- Writes operation logs to CloudWatch Logs.

## Processing flow

```text
AWS IoT Core
→ AWS IoT Rule
→ SmartHomeIngest
→ Amazon DynamoDB
→ Amazon SNS
→ Amazon CloudWatch Logs
```

The AWS IoT Rules are created in section 5.9. In this section, the function is tested manually with a Lambda test event.

---

## 5.8.1. Create the Lambda function

Sign in using:

```text
dinh-fcj
```

Verify the Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

Open:

```text
AWS Lambda
→ Functions
→ Create function
```

Select:

```text
Author from scratch
```

Configure:

| Property | Value |
|---|---|
| Function name | `SmartHomeIngest` |
| Runtime | `Python 3.14` |
| Architecture | `x86_64` |

Under **Change default execution role**, select:

```text
Use an existing role
```

Select:

```text
SmartHomeIngestRole
```

Then select:

```text
Create function
```

![Configure the SmartHomeIngest Lambda function](/images/5.8.1.png)

### Why use an existing role?

`SmartHomeIngestRole` was restricted in section 5.7. It allows the function to:

- Write to `SmartHomeTelemetry`.
- Publish to the SNS topic.
- Write logs to CloudWatch.

---

## 5.8.2. Configure memory and timeout

Open:

```text
SmartHomeIngest
→ Configuration
→ General configuration
→ Edit
```

Configure:

| Property | Value |
|---|---|
| Memory | `128 MB` |
| Timeout | `10 seconds` |
| Ephemeral storage | Keep the default value |

Select:

```text
Save
```

The function processes one JSON message per invocation, so 128 MB is sufficient for this workshop.

A 30-second timeout prevents the function from running indefinitely when DynamoDB or SNS encounters a problem.

---

## 5.8.3. Configure environment variables

Open:

```text
Configuration
→ Environment variables
→ Edit
```

Add:

| Key | Value |
|---|---|
| `TABLE_NAME` | `SmartHomeTelemetry` |
| `RETENTION_DAYS` | `90` |
| `SNS_TOPIC_ARN` | ARN of `SmartHomeAlerts` |

Example:

```text
TABLE_NAME=SmartHomeTelemetry
RETENTION_DAYS=90
SNS_TOPIC_ARN=arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts
```

If the SNS topic has not yet been created, leave `SNS_TOPIC_ARN` empty and update it in section 5.10.

![SmartHomeIngest environment variables](/images/5.8.3.png)

### Why use environment variables?

Environment variables allow the table name, retention period, or SNS topic to be changed without modifying the source code.

Do not store the following as environment variables:

- Passwords.
- Access keys.
- Secret access keys.
- Certificate private keys.


---

## 5.8.4. Add the Python source code

Open the **Code** tab and select:

```text
lambda_function.py
```

Replace the default code with the Python source from the Vietnamese file above, then select:

```text
Deploy
```

The same `lambda_function.py` source is used for both language versions of the workshop.

```python
import logging
import os
import time
from datetime import datetime, timezone
from decimal import Decimal

import boto3
from botocore.exceptions import ClientError


logger = logging.getLogger()
logger.setLevel(logging.INFO)


TABLE_NAME = os.environ.get(
    "TABLE_NAME",
    "SmartHomeTelemetry"
)

RETENTION_DAYS = int(
    os.environ.get("RETENTION_DAYS", "90")
)

SNS_TOPIC_ARN = os.environ.get(
    "SNS_TOPIC_ARN",
    ""
)


dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(TABLE_NAME)
sns = boto3.client("sns")


ALERT_CONFIG = {
    "gas_alarm_started": {
        "subject": "[NGUY HIEM] Phat hien khi gas tai home01",
        "title": "PHAT HIEN KHI GAS"
    },
    "gas_alarm_cleared": {
        "subject": "[AN TOAN] Canh bao khi gas da ket thuc",
        "title": "KHI GAS DA TRO LAI MUC AN TOAN"
    },
    "door_brute_force_detected": {
        "subject": "[CANH BAO] Nhap sai mat khau cua 3 lan",
        "title": "CANH BAO TRUY CAP CUA"
    },
    "test_alert": {
        "subject": "[TEST] Smart Home Alert",
        "title": "KIEM TRA HE THONG CANH BAO"
    }
}


BLOCKED_FIELD_WORDS = (
    "password",
    "passcode",
    "doorpass",
    "secret"
)


def convert_numbers(value):
    """Chuyển float sang Decimal để DynamoDB có thể lưu."""

    if isinstance(value, float):
        return Decimal(str(value))

    if isinstance(value, dict):
        return {
            key: convert_numbers(item)
            for key, item in value.items()
        }

    if isinstance(value, list):
        return [
            convert_numbers(item)
            for item in value
        ]

    return value


def remove_sensitive_fields(value):
    """Loại bỏ trường nhạy cảm kể cả khi nằm trong object lồng nhau."""

    if isinstance(value, dict):
        safe_value = {}

        for key, item in value.items():
            normalized_key = key.lower()

            is_sensitive = any(
                blocked_word in normalized_key
                for blocked_word in BLOCKED_FIELD_WORDS
            )

            if not is_sensitive:
                safe_value[key] = remove_sensitive_fields(item)

        return safe_value

    if isinstance(value, list):
        return [
            remove_sensitive_fields(item)
            for item in value
        ]

    return value


def build_alert_message(event):
    """Tạo nội dung email cảnh báo."""

    event_type = str(event.get("eventType", "unknown"))
    config = ALERT_CONFIG[event_type]

    lines = [
        config["title"],
        "",
        f"Home ID: {event.get('homeId', '-')}",
        f"Device ID: {event.get('deviceId', '-')}",
        f"Gateway ID: {event.get('gatewayId', '-')}",
        f"Event type: {event_type}",
        f"Event ID: {event.get('eventId', '-')}",
        f"Thoi gian thiet bi: {event.get('timestamp', '-')}",
        f"Gas ADC: {event.get('gasRaw', '-')}",
        f"Muc do: {event.get('severity', '-')}",
        f"Noi dung: {event.get('message', '-')}",
        "",
        "Thong bao duoc gui tu AWS Smart Home."
    ]

    return config["subject"], "\n".join(lines)


def publish_alert(event):
    """Chỉ gửi SNS đối với các loại sự kiện được cho phép."""

    event_type = str(event.get("eventType", "")).strip()

    if event_type not in ALERT_CONFIG:
        return {
            "required": False,
            "sent": False
        }

    if not SNS_TOPIC_ARN:
        logger.error(
            "SNS_TOPIC_ARN environment variable is missing"
        )

        return {
            "required": True,
            "sent": False,
            "error": "SNS_TOPIC_ARN is missing"
        }

    subject, message = build_alert_message(event)

    try:
        response = sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=subject,
            Message=message
        )

        sns_message_id = response.get("MessageId")

        logger.info(
            "SNS alert sent: eventType=%s messageId=%s",
            event_type,
            sns_message_id
        )

        return {
            "required": True,
            "sent": True,
            "snsMessageId": sns_message_id
        }

    except ClientError as error:
        logger.exception(
            "Failed to publish SNS alert: eventType=%s",
            event_type
        )

        return {
            "required": True,
            "sent": False,
            "error": error.response["Error"].get(
                "Code",
                "UnknownError"
            )
        }


def lambda_handler(event, context):
    """Lưu telemetry/event và gửi SNS cho sự kiện cảnh báo."""

    if not isinstance(event, dict):
        logger.warning(
            "Rejected event: payload is not a JSON object"
        )

        return {
            "statusCode": 400,
            "message": "Payload must be a JSON object"
        }

    safe_event = remove_sensitive_fields(event)

    home_id = str(
        safe_event.get("homeId", "")
    ).strip()

    device_id = str(
        safe_event.get("deviceId", "")
    ).strip()

    timestamp = str(
        safe_event.get("timestamp", "")
    ).strip()

    is_event = bool(
        safe_event.get("eventType")
    )

    record_type = (
        "event" if is_event else "telemetry"
    )

    if is_event:
        record_id = str(
            safe_event.get("eventId", "")
        ).strip()
    else:
        record_id = str(
            safe_event.get("messageId", "")
        ).strip()

    missing_fields = []

    if not home_id:
        missing_fields.append("homeId")

    if not device_id:
        missing_fields.append("deviceId")

    if not timestamp:
        missing_fields.append("timestamp")

    if not record_id:
        missing_fields.append(
            "eventId" if is_event else "messageId"
        )

    if missing_fields:
        logger.warning(
            "Rejected %s: missing fields %s",
            record_type,
            ",".join(missing_fields)
        )

        return {
            "statusCode": 400,
            "message": "Missing required fields",
            "missingFields": missing_fields
        }

    device_key = f"{home_id}#{device_id}"
    record_key = f"{timestamp}#{record_id}"

    current_time = datetime.now(timezone.utc)

    expires_at = (
        int(time.time())
        + RETENTION_DAYS * 24 * 60 * 60
    )

    item = convert_numbers(safe_event)

    item.update({
        "deviceKey": device_key,
        "recordKey": record_key,
        "recordType": record_type,
        "ingestedAt": current_time.isoformat(),
        "expiresAt": expires_at
    })

    try:
        table.put_item(
            Item=item,
            ConditionExpression=(
                "attribute_not_exists(deviceKey) "
                "AND attribute_not_exists(recordKey)"
            )
        )

        logger.info(
            "Stored %s: device=%s record=%s",
            record_type,
            device_key,
            record_id
        )

    except ClientError as error:
        error_code = error.response["Error"]["Code"]

        if error_code == "ConditionalCheckFailedException":
            logger.info(
                "Duplicate ignored: %s",
                record_id
            )

            return {
                "statusCode": 200,
                "stored": False,
                "duplicate": True,
                "recordType": record_type,
                "recordId": record_id,
                "alertSent": False
            }

        logger.exception(
            "Failed to write data to DynamoDB"
        )

        raise

    alert_result = publish_alert(safe_event)

    return {
        "statusCode": 201,
        "stored": True,
        "duplicate": False,
        "recordType": record_type,
        "recordId": record_id,
        "alertRequired": alert_result.get(
            "required",
            False
        ),
        "alertSent": alert_result.get(
            "sent",
            False
        ),
        "snsMessageId": alert_result.get(
            "snsMessageId"
        ),
        "alertError": alert_result.get(
            "error"
        )
    }
```
![Mã nguồn SmartHomeIngest đã được deploy](/images/5.8.4.png)
---

## 5.8.5. Source code explanation

### Incoming event

Lambda receives the JSON payload through:

```python
event
```

The event is later passed to the function by an AWS IoT Rule.

### Floating-point conversion

Boto3 does not write Python `float` values directly to DynamoDB. The function converts them to `Decimal` before storage.

### DynamoDB keys

The partition key combines the home and device:

```text
home01#esp32-01
```

The sort key combines:

```text
Record type + timestamp + messageId
```

Example:

```text
TELEMETRY#2026-09-14T10:30:00Z#telemetry-001
```

### TTL generation

With `RETENTION_DAYS=90`, the function generates an `expiresAt` value 90 days in the future as Unix epoch seconds.

### Duplicate prevention

The DynamoDB `ConditionExpression` writes the item only when the same composite key does not already exist.

A duplicate invocation returns:

```json
{
  "statusCode": 200,
  "duplicate": true
}
```

A duplicate event is not stored again and does not generate another SNS email.

### Alert publishing

SNS is invoked only for:

```text
gas_alarm_started
gas_alarm_cleared
door_brute_force_detected
```

Regular telemetry does not generate an email notification.

---

## 5.8.6. Create a telemetry test event

In the Lambda Console, select:

```text
Test
→ Create new event
```

Enter:

```text
TestTelemetry
```

Use:

```json
{
  "messageId": "test-telemetry-001",
  "recordType": "telemetry",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "timestamp": "2026-09-14T10:30:00Z",
  "temperature": 28.5,
  "humidity": 67.2,
  "gasRaw": 1048,
  "doorState": "CLOSED",
  "lightState": "OFF",
  "fanState": "OFF"
}
```

Select:

```text
Save
→ Test
```

Expected result:

```text
Status: Succeeded
statusCode: 200
duplicate: false
recordType: telemetry
```

![Successful Lambda telemetry test](/images/5.8.6.png)

> Change `messageId` and `timestamp` when a new record is required.

---

## 5.8.7. Test duplicate prevention

Run `TestTelemetry` again without changing the payload.

The second invocation should return:

```json
{
  "statusCode": 200,
  "duplicate": true,
  "messageId": "test-telemetry-001"
}
```

This confirms that the same message is not stored more than once.

Deduplication is important because MQTT QoS 1 and service retry behavior can deliver the same message more than once.

---

## 5.8.8. Verify the DynamoDB record

Open:

```text
DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
→ Run
```

Find:

```text
messageId = test-telemetry-001
```

Verify that:

- `deviceKey` is `home01#esp32-01`.
- `recordType` is `telemetry`.
- Temperature, humidity, and gas ADC are available.
- `expiresAt` uses the Number data type.
- The second test did not create a duplicate item.

![DynamoDB item created by SmartHomeIngest](/images/5.8.8.png)

---

## Configuration summary

| Property | Value |
|---|---|
| Function name | `SmartHomeIngest` |
| Runtime | `Python 3.14` |
| Architecture | `x86_64` |
| Memory | `128 MB` |
| Timeout | `10 seconds` |
| Execution role | `SmartHomeIngestRole` |
| DynamoDB table | `SmartHomeTelemetry` |
| RETENTION_DAYS | `90 days` |
| Handler | `lambda_function.lambda_handler` |

---

## Completion checklist

- [ ] `SmartHomeIngest` has been created.
- [ ] The runtime is Python 3.14.
- [ ] The execution role is `SmartHomeIngestRole`.
- [ ] Environment variables are configured.
- [ ] The source code has been deployed.
- [ ] The telemetry test has succeeded.
- [ ] The record appears in DynamoDB.
- [ ] `expiresAt` uses Unix epoch seconds as a Number.
- [ ] The second test returns `duplicate: true`.
- [ ] Telemetry does not invoke SNS.
- [ ] Logs do not contain passwords or secret information.

## Conclusion

In this section, we created the `SmartHomeIngest` Lambda function to normalize, store, and deduplicate IoT data.

The function was manually tested and successfully wrote data to `SmartHomeTelemetry`.

Next, we will create AWS IoT Rules to invoke the function automatically when AWS IoT Core receives telemetry or events.

## References

- [AWS – Create a Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
- [AWS – Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)
- [AWS – Testing Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/testing-functions.html)
- [AWS – DynamoDB condition expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html)