---
title: "Xây dựng Lambda SmartHomeIngest"
date: 2026-09-14
weight: 8
chapter: false
pre: "<b>5.8. </b>"
---

# Xây dựng Lambda SmartHomeIngest

Trong phần này, chúng ta sẽ tạo Lambda function `SmartHomeIngest` để xử lý dữ liệu telemetry và events nhận từ AWS IoT Core.

Function thực hiện các nhiệm vụ:

- Kiểm tra dữ liệu đầu vào.
- Phân biệt telemetry và event.
- Tạo khóa lưu trữ DynamoDB.
- Chuyển số thực sang kiểu dữ liệu tương thích DynamoDB.
- Tạo thời điểm hết hạn cho TTL.
- Chống ghi trùng bản tin.
- Gửi cảnh báo SNS đối với sự kiện quan trọng.
- Ghi nhật ký vào CloudWatch Logs.

## Luồng xử lý

```text
AWS IoT Core
→ AWS IoT Rule
→ SmartHomeIngest
→ Amazon DynamoDB
→ Amazon SNS
→ Amazon CloudWatch Logs
```

AWS IoT Rule sẽ được tạo trong phần 5.9. Ở phần này, Lambda được kiểm thử thủ công bằng test event.

---

## 5.8.1. Tạo Lambda function

Đăng nhập bằng IAM user:

```text
dinh-fcj
```

Kiểm tra Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

Mở:

```text
AWS Lambda
→ Functions
→ Create function
```

Chọn:

```text
Author from scratch
```

Cấu hình:

| Thuộc tính | Giá trị |
|---|---|
| Function name | `SmartHomeIngest` |
| Runtime | `Python 3.14` |
| Architecture | `x86_64` |

Trong phần **Change default execution role**, chọn:

```text
Use an existing role
```

Chọn execution role:

```text
SmartHomeIngestRole
```

Sau đó chọn:

```text
Create function
```

![Cấu hình Lambda SmartHomeIngest](/images/5.8.1.png)

### Tại sao sử dụng existing role?

`SmartHomeIngestRole` đã được giới hạn quyền trong phần 5.7. Việc sử dụng role có sẵn giúp kiểm soát chính xác Lambda được phép:

- Ghi vào bảng `SmartHomeTelemetry`.
- Publish đến SNS topic.
- Ghi log vào CloudWatch.

---

## 5.8.2. Cấu hình bộ nhớ và timeout

Mở:

```text
SmartHomeIngest
→ Configuration
→ General configuration
→ Edit
```

Cấu hình:

| Thuộc tính | Giá trị |
|---|---|
| Memory | `128 MB` |
| Timeout | `10 seconds` |
| Ephemeral storage | Giữ mặc định |

Chọn:

```text
Save
```

Function chỉ xử lý một bản tin JSON trong mỗi lần gọi nên 128 MB là đủ cho phạm vi Workshop.

Timeout 30 giây giúp tránh function chạy quá lâu khi DynamoDB hoặc SNS gặp sự cố.

---

## 5.8.3. Cấu hình biến môi trường

Mở:

```text
Configuration
→ Environment variables
→ Edit
```

Thêm các biến:

| Key | Value |
|---|---|
| `TABLE_NAME` | `SmartHomeTelemetry` |
| `RETENTION_DAYS` | `90` |
| `SNS_TOPIC_ARN` | ARN của SNS topic `SmartHomeAlerts` |

Ví dụ:

```text
TABLE_NAME=SmartHomeTelemetry
RETENTION_DAYS=90
SNS_TOPIC_ARN=arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts
```

Nếu SNS topic chưa được tạo, có thể để `SNS_TOPIC_ARN` trống và cập nhật lại trong phần 5.10.

![Biến môi trường của SmartHomeIngest](/images/5.8.3.png)

### Tại sao sử dụng biến môi trường?

Biến môi trường cho phép thay đổi tên bảng, thời gian lưu dữ liệu hoặc SNS topic mà không phải sửa mã nguồn.

Không sử dụng biến môi trường để lưu:

- Mật khẩu.
- Access key.
- Secret access key.
- Certificate private key.


---

## 5.8.4. Thêm mã nguồn Python

Mở tab:

```text
Code
```

Chọn file:

```text
lambda_function.py
```

Xóa mã mặc định và thay bằng:

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

Chọn:

```text
Deploy
```

![Mã nguồn SmartHomeIngest đã được deploy](/images/5.8.4.png)

---

## 5.8.5. Giải thích mã nguồn

### Xử lý dữ liệu đầu vào

Lambda nhận dữ liệu thông qua biến:

```python
event
```

`event` là payload JSON được AWS IoT Rule chuyển đến function.

### Chuyển đổi số thực

Boto3 không ghi trực tiếp kiểu `float` của Python vào DynamoDB. Hàm:

```python
convert_for_dynamodb()
```

chuyển `float` thành `Decimal` trước khi lưu.

### Tạo khóa DynamoDB

```python
device_key = f"{home_id}#{device_id}"
```

Ví dụ:

```text
home01#esp32-01
```

`recordKey` kết hợp:

```text
Loại bản tin + timestamp + messageId
```

Ví dụ:

```text
TELEMETRY#2026-09-14T10:30:00Z#telemetry-001
```

### Tạo TTL

```python
expires_at = int(time.time()) + RETENTION_DAYS * 24 * 60 * 60
```

Với `RETENTION_DAYS=90`, bản ghi sẽ đủ điều kiện được DynamoDB xóa sau 90 ngày.

### Chống ghi trùng

`ConditionExpression` chỉ cho phép ghi nếu bản ghi có cùng khóa chưa tồn tại.

Nếu cùng một bản tin được gửi lại, Lambda trả về:

```json
{
  "statusCode": 200,
  "duplicate": true
}
```

Bản tin trùng không được ghi thêm và không tạo email SNS thứ hai.

### Gửi cảnh báo

SNS chỉ được gọi đối với các event type:

```text
gas_alarm_started
gas_alarm_cleared
door_brute_force_detected
```

Telemetry thông thường không gửi email.

---

## 5.8.6. Tạo test event telemetry

Trong Lambda Console, chọn:

```text
Test
→ Create new event
```

Đặt tên:

```text
TestTelemetry
```

Nhập:

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

Chọn:

```text
Save
→ Test
```

Kết quả thành công cần có:

```text
Status: Succeeded
statusCode: 200
duplicate: false
recordType: telemetry
```

![Lambda telemetry test thành công](/images/5.8.6.png)

> Khi kiểm thử lại, hãy thay đổi `messageId` và `timestamp` nếu muốn tạo bản ghi mới.

---

## 5.8.7. Kiểm tra chống dữ liệu trùng

Chạy lại test event `TestTelemetry` mà không thay đổi payload.

Lần chạy thứ hai cần trả về:

```json
{
  "statusCode": 200,
  "duplicate": true,
  "messageId": "test-telemetry-001"
}
```

Điều này chứng minh cùng một bản tin không được lưu nhiều lần.

Cơ chế chống trùng đặc biệt quan trọng vì MQTT QoS 1 và cơ chế retry của dịch vụ có thể làm một bản tin được chuyển lại.

---

## 5.8.8. Kiểm tra dữ liệu DynamoDB

Mở:

```text
DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
→ Run
```

Tìm bản ghi có:

```text
messageId = test-telemetry-001
```

Kiểm tra:

- `deviceKey` là `home01#esp32-01`.
- `recordType` là `telemetry`.
- Có nhiệt độ, độ ẩm và gas ADC.
- `expiresAt` có kiểu Number.
- Không có bản ghi trùng sau lần test thứ hai.

![Bản ghi do SmartHomeIngest tạo trong DynamoDB](/images/5.8.8.png)

---

## Cấu hình tổng hợp

| Thuộc tính | Giá trị |
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

## Kiểm tra hoàn thành

- [ ] `SmartHomeIngest` đã được tạo.
- [ ] Runtime là Python 3.14.
- [ ] Execution role là `SmartHomeIngestRole`.
- [ ] Biến môi trường đã được cấu hình.
- [ ] Mã nguồn đã được Deploy.
- [ ] Telemetry test có trạng thái `Succeeded`.
- [ ] Bản ghi đã xuất hiện trong DynamoDB.
- [ ] `expiresAt` là Number theo Unix epoch giây.
- [ ] Test lần hai trả về `duplicate: true`.
- [ ] Telemetry không kích hoạt SNS.
- [ ] Log không chứa mật khẩu hoặc thông tin bí mật.

## Kết luận

Trong phần này, chúng ta đã tạo Lambda `SmartHomeIngest` để chuẩn hóa, lưu trữ và chống trùng dữ liệu IoT.

Function đã được kiểm thử thủ công và ghi thành công dữ liệu vào `SmartHomeTelemetry`.

Tiếp theo, chúng ta sẽ tạo AWS IoT Rules để tự động gọi Lambda khi AWS IoT Core nhận telemetry hoặc events.

## Tài liệu tham khảo

- [AWS – Create a Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
- [AWS – Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)
- [AWS – Testing Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/testing-functions.html)
- [AWS – DynamoDB condition expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html)