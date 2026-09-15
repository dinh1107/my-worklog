---
title: "Tạo Lambda xuất báo cáo"
date: 2026-09-14
weight: 14
chapter: false
pre: "<b>5.14. </b>"
---

# Tạo Lambda xuất báo cáo JSON và CSV

Trong phần trước, chúng ta đã tạo S3 bucket để lưu báo cáo. Tiếp theo, chúng ta sẽ tạo Lambda `SmartHomeExport` để:

1. Truy vấn dữ liệu từ DynamoDB.
2. Lọc dữ liệu theo ngày.
3. Tính thống kê telemetry.
4. Thống kê số lượng sự kiện.
5. Tạo báo cáo JSON và CSV.
6. Ghi hai file vào Amazon S3.

Luồng xử lý:

```text
DynamoDB SmartHomeTelemetry
→ SmartHomeExport
→ report.json và report.csv
→ Amazon S3
```

## 5.14.1. Thông tin cấu hình

| Thành phần | Giá trị |
|---|---|
| Function name | `SmartHomeExport` |
| Runtime | Python 3.14 |
| Architecture | x86_64 |
| Execution role | `SmartHomeExportRole` |
| DynamoDB table | `SmartHomeTelemetry` |
| S3 bucket | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |
| Memory | 128 MB |
| Timeout | 30 giây |
| Region | `ap-southeast-1` |

Thay `<ACCOUNT_ID>` bằng AWS account ID thực tế.

## 5.14.2. Tạo execution role cho Lambda

Thực hiện bước này bằng tài khoản root hoặc tài khoản quản trị.

Mở:

```text
IAM
→ Roles
→ Create role
```

Trong phần **Trusted entity type**, chọn:

```text
AWS service
```

Trong phần **Use case**, chọn:

```text
Lambda
```

Chọn **Next** và gắn managed policy:

```text
AWSLambdaBasicExecutionRole
```

Policy này cho phép Lambda ghi log vào CloudWatch.

Đặt tên role:

```text
SmartHomeExportRole
```

Sau đó chọn:

```text
Create role
```

## 5.14.3. Cấp quyền DynamoDB và S3

Mở role vừa tạo:

```text
IAM
→ Roles
→ SmartHomeExportRole
→ Permissions
→ Add permissions
→ Create inline policy
→ JSON
```

Nhập policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSmartHomeTelemetry",
      "Effect": "Allow",
      "Action": [
        "dynamodb:Query",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:ap-southeast-1:<ACCOUNT_ID>:table/SmartHomeTelemetry"
    },
    {
      "Sid": "WriteSmartHomeReports",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1/reports/*"
    },
    {
      "Sid": "ViewReportBucketLocation",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1"
    }
  ]
}
```

Đặt tên policy:

```text
SmartHomeExportDataPolicy
```

Sau đó chọn:

```text
Create policy
```

Role này chỉ có quyền:

- Đọc dữ liệu từ `SmartHomeTelemetry`.
- Ghi object vào prefix `reports/`.
- Ghi log vào CloudWatch.

Role không có quyền xóa bảng, xóa bucket hoặc công khai báo cáo.

## 5.14.4. Tạo Lambda function

Đăng xuất tài khoản root và đăng nhập lại bằng IAM user:

```text
dinh-fcj
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

Nhập các giá trị:

| Thuộc tính | Giá trị |
|---|---|
| Function name | `SmartHomeExport` |
| Runtime | `Python 3.14` |
| Architecture | `x86_64` |

Trong phần **Change default execution role**, chọn:

```text
Use an existing role
```

Chọn role:

```text
SmartHomeExportRole
```

Sau đó chọn:

```text
Create function
```

Nếu xuất hiện lỗi `iam:PassRole`, tài khoản quản trị cần cho phép `dinh-fcj` truyền riêng role `SmartHomeExportRole` cho dịch vụ Lambda.

![Lambda SmartHomeExport](/images/5.14.4.png)



## 5.14.5. Cấu hình biến môi trường

Trước hết, mở một item trong bảng DynamoDB và sao chép chính xác giá trị:

```text
deviceKey
```

Sau đó mở:

```text
SmartHomeExport
→ Configuration
→ Environment variables
→ Edit
```

Thêm các biến:

| Key | Value |
|---|---|
| `TABLE_NAME` | `SmartHomeTelemetry` |
| `REPORT_BUCKET` | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |
| `DEVICE_KEY` | Giá trị `deviceKey` thực tế trong DynamoDB |
| `HOME_ID` | `home01` |
| `DEVICE_ID` | `esp32-01` |

Ví dụ, nếu item trong DynamoDB sử dụng:

```text
HOME#home01#DEVICE#esp32-01
```

thì đặt:

```text
DEVICE_KEY=HOME#home01#DEVICE#esp32-01
```

Không tự suy đoán định dạng `deviceKey`; phải sao chép từ item đã được `SmartHomeIngest` tạo.

Chọn:

```text
Save
```

## 5.14.6. Cấu hình timeout

Mở:

```text
SmartHomeExport
→ Configuration
→ General configuration
→ Edit
```

Đặt:

| Thuộc tính | Giá trị |
|---|---|
| Memory | `256 MB` |
| Timeout | `30 seconds` |

Chọn:

```text
Save
```

Timeout 30 giây phù hợp với lượng dữ liệu của workshop. Nếu bảng có lượng dữ liệu lớn, cần tối ưu DynamoDB key hoặc tăng timeout.

## 5.14.7. Thêm mã nguồn

Mở tab:

```text
Code
→ lambda_function.py
```

Thay toàn bộ mã mặc định bằng:

```python
import csv
import json
import logging
import os
from datetime import date, datetime, timedelta, timezone
from decimal import Decimal
from io import StringIO
from zoneinfo import ZoneInfo

import boto3
from boto3.dynamodb.conditions import Key

logger = logging.getLogger()
logger.setLevel(logging.INFO)

dynamodb = boto3.resource("dynamodb")
s3 = boto3.client("s3")

TABLE_NAME = os.environ["TABLE_NAME"]
REPORT_BUCKET = os.environ["REPORT_BUCKET"]
DEVICE_KEY = os.environ["DEVICE_KEY"]
HOME_ID = os.environ.get("HOME_ID", "home01")
DEVICE_ID = os.environ.get("DEVICE_ID", "esp32-01")

table = dynamodb.Table(TABLE_NAME)
VN_TIMEZONE = ZoneInfo("Asia/Ho_Chi_Minh")


def convert_decimal(value):
    if isinstance(value, Decimal):
        if value % 1 == 0:
            return int(value)
        return float(value)

    if isinstance(value, list):
        return [convert_decimal(item) for item in value]

    if isinstance(value, dict):
        return {
            key: convert_decimal(item)
            for key, item in value.items()
        }

    return value


def parse_timestamp(item):
    value = (
        item.get("timestamp")
        or item.get("eventTime")
        or item.get("sentAt")
        or item.get("receivedAt")
    )

    if value is None:
        return None

    if isinstance(value, Decimal):
        value = float(value)

    if isinstance(value, (int, float)):
        if value > 10_000_000_000:
            value = value / 1000

        return datetime.fromtimestamp(value, tz=timezone.utc)

    if isinstance(value, str):
        normalized = value.strip().replace("Z", "+00:00")
        parsed = datetime.fromisoformat(normalized)

        if parsed.tzinfo is None:
            parsed = parsed.replace(tzinfo=VN_TIMEZONE)

        return parsed

    return None


def read_device_items():
    items = []
    query_arguments = {
        "KeyConditionExpression": Key("deviceKey").eq(DEVICE_KEY)
    }

    while True:
        response = table.query(**query_arguments)
        items.extend(response.get("Items", []))

        last_key = response.get("LastEvaluatedKey")
        if not last_key:
            break

        query_arguments["ExclusiveStartKey"] = last_key

    return items


def filter_items_by_date(items, report_date):
    selected = []

    for item in items:
        timestamp = parse_timestamp(item)

        if timestamp is None:
            continue

        local_date = timestamp.astimezone(VN_TIMEZONE).date()

        if local_date == report_date:
            selected.append(item)

    return selected


def numeric_values(items, field_name):
    values = []

    for item in items:
        value = item.get(field_name)

        if isinstance(value, Decimal):
            value = float(value)

        if isinstance(value, (int, float)):
            values.append(float(value))

    return values


def calculate_statistics(values):
    if not values:
        return {
            "minimum": None,
            "maximum": None,
            "average": None
        }

    return {
        "minimum": round(min(values), 2),
        "maximum": round(max(values), 2),
        "average": round(sum(values) / len(values), 2)
    }


def count_events(event_items):
    result = {}

    for item in event_items:
        event_type = item.get("eventType", "unknown")
        result[event_type] = result.get(event_type, 0) + 1

    return result


def create_csv(items):
    output = StringIO()

    fields = [
        "timestamp",
        "recordType",
        "messageId",
        "homeId",
        "deviceId",
        "temperature",
        "humidity",
        "gasRaw",
        "eventType",
        "severity",
        "doorState",
        "lightState",
        "fanState"
    ]

    writer = csv.DictWriter(output, fieldnames=fields)
    writer.writeheader()

    for original_item in items:
        item = convert_decimal(original_item)

        writer.writerow({
            field: item.get(field, "")
            for field in fields
        })

    return output.getvalue()


def get_report_date(event):
    requested_date = event.get("reportDate")

    if requested_date:
        return date.fromisoformat(requested_date)

    return (datetime.now(VN_TIMEZONE) - timedelta(days=1)).date()


def lambda_handler(event, context):
    event = event or {}
    report_date = get_report_date(event)

    logger.info(
        "Creating report for date=%s deviceKey=%s",
        report_date.isoformat(),
        DEVICE_KEY
    )

    all_items = read_device_items()
    report_items = filter_items_by_date(all_items, report_date)

    telemetry_items = [
        item for item in report_items
        if item.get("recordType") == "telemetry"
    ]

    event_items = [
        item for item in report_items
        if item.get("recordType") == "event"
    ]

    report = {
        "reportDate": report_date.isoformat(),
        "timezone": "Asia/Ho_Chi_Minh",
        "homeId": HOME_ID,
        "deviceId": DEVICE_ID,
        "deviceKey": DEVICE_KEY,
        "summary": {
            "totalRecords": len(report_items),
            "telemetryRecords": len(telemetry_items),
            "eventRecords": len(event_items)
        },
        "telemetryStatistics": {
            "temperature": calculate_statistics(
                numeric_values(telemetry_items, "temperature")
            ),
            "humidity": calculate_statistics(
                numeric_values(telemetry_items, "humidity")
            ),
            "gasRaw": calculate_statistics(
                numeric_values(telemetry_items, "gasRaw")
            )
        },
        "eventCounts": count_events(event_items),
        "records": convert_decimal(report_items)
    }

    prefix = (
        f"reports/{HOME_ID}/{DEVICE_ID}/"
        f"{report_date:%Y/%m/%d}"
    )

    json_key = f"{prefix}/report.json"
    csv_key = f"{prefix}/report.csv"

    json_body = json.dumps(
        report,
        ensure_ascii=False,
        indent=2
    )

    csv_body = create_csv(report_items)

    s3.put_object(
        Bucket=REPORT_BUCKET,
        Key=json_key,
        Body=json_body.encode("utf-8"),
        ContentType="application/json",
        ServerSideEncryption="AES256"
    )

    s3.put_object(
        Bucket=REPORT_BUCKET,
        Key=csv_key,
        Body=csv_body.encode("utf-8-sig"),
        ContentType="text/csv; charset=utf-8",
        ServerSideEncryption="AES256"
    )

    result = {
        "statusCode": 200,
        "reportDate": report_date.isoformat(),
        "totalRecords": len(report_items),
        "telemetryRecords": len(telemetry_items),
        "eventRecords": len(event_items),
        "jsonKey": json_key,
        "csvKey": csv_key
    }

    logger.info("Report created: %s", json.dumps(result))
    return result
```

Chọn:

```text
Deploy
```

## 5.14.8. Giải thích mã nguồn

### Lựa chọn ngày báo cáo

Nếu event có:

```json
{
  "reportDate": "2026-09-14"
}
```

Lambda xuất báo cáo cho ngày được chỉ định.

Nếu event là:

```json
{}
```

Lambda tự động lấy ngày hôm trước theo múi giờ:

```text
Asia/Ho_Chi_Minh
```

Cách này sẽ được EventBridge Scheduler sử dụng ở phần tiếp theo.

### Truy vấn DynamoDB

Lambda sử dụng:

```python
Key("deviceKey").eq(DEVICE_KEY)
```

để truy vấn toàn bộ bản ghi của thiết bị. Sau đó, các item được lọc theo ngày trong Python.

Cách này phù hợp với lượng dữ liệu nhỏ của workshop. Với hệ thống lớn, nên thiết kế sort key hoặc Global Secondary Index để truy vấn trực tiếp theo khoảng thời gian.

### Thống kê telemetry

Lambda tính ba giá trị:

```text
minimum
maximum
average
```

cho:

- Nhiệt độ.
- Độ ẩm.
- Giá trị gas ADC thô.

`gasRaw` không được chuyển thành ppm vì cảm biến chưa được hiệu chuẩn bằng khí chuẩn.

### Thống kê sự kiện

Các sự kiện được đếm theo `eventType`, ví dụ:

```json
{
  "gas_alarm_started": 1,
  "gas_alarm_cleared": 1,
  "door_brute_force_detected": 2
}
```

### Tạo object S3

Hai file được ghi vào:

```text
reports/home01/esp32-01/YYYY/MM/DD/report.json
reports/home01/esp32-01/YYYY/MM/DD/report.csv
```

JSON phù hợp với ứng dụng và xử lý tự động. CSV phù hợp để mở bằng Excel hoặc phần mềm bảng tính.

## 5.14.9. Tạo test event

Mở:

```text
SmartHomeExport
→ Test
→ Create new event
```

Đặt tên:

```text
ExportSpecificDate
```

Nhập:

```json
{
  "reportDate": "2026-09-14"
}
```

Chọn:

```text
Save
→ Test
```

Hãy chọn một ngày chắc chắn có dữ liệu trong DynamoDB.

Kết quả thành công có dạng:

```json
{
  "statusCode": 200,
  "reportDate": "2026-09-14",
  "totalRecords": 120,
  "telemetryRecords": 117,
  "eventRecords": 3,
  "jsonKey": "reports/home01/esp32-01/2026/09/14/report.json",
  "csvKey": "reports/home01/esp32-01/2026/09/14/report.csv"
}
```

Số lượng bản ghi thực tế có thể khác ví dụ.

![Kiểm thử SmartHomeExport thành công](/images/5.14.9.png)



## 5.14.10. Kiểm tra file trong S3

Mở:

```text
Amazon S3
→ smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

Đi đến:

```text
reports/
→ home01/
→ esp32-01/
→ 2026/
→ 09/
→ 14/
```

Kết quả phải có:

```text
report.json
report.csv
```

Kiểm tra:

- Cả hai file có kích thước lớn hơn `0 B`.
- `Last modified` khớp với thời điểm Lambda chạy.
- Content type của JSON là `application/json`.
- Có thể tải file về bằng nút **Download**.

![Các file báo cáo trong S3](/images/5.14.10.png)



## 5.14.11. Kiểm tra nội dung báo cáo

Nội dung `report.json` có cấu trúc:

```json
{
  "reportDate": "2026-09-14",
  "timezone": "Asia/Ho_Chi_Minh",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "summary": {
    "totalRecords": 120,
    "telemetryRecords": 117,
    "eventRecords": 3
  },
  "telemetryStatistics": {
    "temperature": {
      "minimum": 27.5,
      "maximum": 31.2,
      "average": 29.4
    },
    "humidity": {
      "minimum": 65,
      "maximum": 76,
      "average": 71.2
    },
    "gasRaw": {
      "minimum": 980,
      "maximum": 2050,
      "average": 1085.4
    }
  }
}
```

Các số liệu trên chỉ là ví dụ. Website báo cáo nên sử dụng kết quả thực tế của project.

File `report.csv` có các cột:

```text
timestamp,recordType,messageId,homeId,deviceId,temperature,
humidity,gasRaw,eventType,severity,doorState,lightState,fanState
```

## 5.14.12. Kiểm tra S3 Versioning

Chạy lại cùng test event:

```json
{
  "reportDate": "2026-09-14"
}
```

Sau đó mở thư mục chứa báo cáo và bật:

```text
Show versions
```

Do bucket đã bật Versioning, mỗi file sẽ có thêm một version mới thay vì làm mất hoàn toàn version trước.



## 5.14.13. Một số lỗi thường gặp

### Lambda báo AccessDenied khi Query

Kiểm tra:

- Execution role là `SmartHomeExportRole`.
- Role có `dynamodb:Query`.
- ARN trỏ đúng bảng `SmartHomeTelemetry`.
- Region là `ap-southeast-1`.

### Lambda báo AccessDenied khi ghi S3

Kiểm tra:

- Role có `s3:PutObject`.
- Resource kết thúc bằng:

```text
/reports/*
```

- `REPORT_BUCKET` đúng với tên bucket thực tế.

### Báo cáo có 0 bản ghi

Kiểm tra:

- `DEVICE_KEY` khớp chính xác với item trong DynamoDB.
- `reportDate` là ngày có dữ liệu.
- Timestamp của item có định dạng hợp lệ.
- Múi giờ của dữ liệu được hiểu đúng.

Báo cáo có 0 bản ghi vẫn có thể được tạo thành công; đây không phải lỗi ghi S3.

### File CSV lỗi tiếng Việt khi mở bằng Excel

Mã nguồn sử dụng:

```python
utf-8-sig
```

để hỗ trợ Excel nhận diện UTF-8. Nếu vẫn lỗi, hãy nhập file bằng chức năng **Data → From Text/CSV** và chọn UTF-8.

### Lambda timeout

Nếu dữ liệu quá nhiều:

- Tăng timeout tạm thời.
- Giảm khoảng dữ liệu.
- Tối ưu sort key để Query theo khoảng ngày.
- Không chuyển sang `Scan` toàn bảng.

## 5.14.14. Kết quả đạt được

Sau khi hoàn thành phần này:

- `SmartHomeExportRole` đã được tạo theo nguyên tắc quyền tối thiểu.
- Lambda `SmartHomeExport` đã được triển khai.
- Hàm có thể chọn ngày báo cáo từ event.
- Payload `{}` tự động chọn ngày hôm trước theo giờ Việt Nam.
- Dữ liệu DynamoDB được tổng hợp thành JSON và CSV.
- Hai file được lưu đúng cấu trúc trong S3.
- S3 Versioning giữ lại các phiên bản báo cáo cũ.
- CloudWatch ghi lại kết quả mỗi lần xuất báo cáo.

## Tài liệu tham khảo

- [Creating an AWS Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
- [Querying DynamoDB tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html)
- [Amazon S3 PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html)
- [Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)