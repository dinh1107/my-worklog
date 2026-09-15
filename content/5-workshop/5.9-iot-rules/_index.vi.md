---
title: "Tạo AWS IoT Rules"
date: 2026-09-14
weight: 9
chapter: false
pre: "<b>5.9. </b>"
---

# Tạo AWS IoT Rules để gọi Lambda

Ở bước trước, chúng ta đã tạo hàm Lambda `SmartHomeIngest` để xử lý dữ liệu nhà thông minh. Tuy nhiên, Lambda chưa tự động nhận được các bản tin MQTT gửi đến AWS IoT Core.

Trong phần này, chúng ta sẽ tạo hai AWS IoT Rule:

- `SmartHomeTelemetryRule`: tiếp nhận dữ liệu trạng thái định kỳ.
- `SmartHomeEventsRule`: tiếp nhận các sự kiện cần xử lý ngay.

Cả hai rule đều chuyển payload đến hàm Lambda `SmartHomeIngest`.

## 5.9.1. Vai trò của AWS IoT Rules Engine

AWS IoT Rules Engine kiểm tra các bản tin được publish lên MQTT topic. Khi topic khớp với câu lệnh SQL của rule, AWS IoT Core thực hiện action đã cấu hình.

Trong workshop này, action là gọi hàm Lambda:

```text
MQTT topic → AWS IoT Rule → SmartHomeIngest → DynamoDB
```

Hai khái niệm sau có vai trò khác nhau:

| Thành phần | Vai trò |
|---|---|
| AWS IoT Policy | Cho phép thiết bị kết nối, publish hoặc subscribe MQTT |
| AWS IoT Rule | Chọn và chuyển bản tin MQTT đến dịch vụ AWS |

Chúng ta sử dụng hai rule riêng để Lambda nhận biết rõ bản tin nào là `telemetry` và bản tin nào là `event`.

## 5.9.2. Cấu hình cần sử dụng

Kiểm tra Region trước khi thực hiện:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Các tài nguyên được sử dụng:

| Thành phần | Giá trị |
|---|---|
| Lambda function | `SmartHomeIngest` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| IoT SQL version | `2016-03-23` |
| Telemetry rule | `SmartHomeTelemetryRule` |
| Events rule | `SmartHomeEventsRule` |

Hãy đăng nhập bằng IAM user `dinh-fcj`. Nếu xuất hiện lỗi `AccessDenied`, cần dùng tài khoản root hoặc quản trị để kiểm tra lại quyền của nhóm `SmartHome-Developers`.

## 5.9.3. Tạo rule tiếp nhận telemetry

### Bước 1: Mở trang AWS IoT Rules

Trong AWS Management Console, mở:

```text
AWS IoT Core
→ Message routing
→ Rules
```

Chọn:

```text
Create rule
```

### Bước 2: Nhập thông tin rule

Nhập các giá trị sau:

| Thuộc tính | Giá trị |
|---|---|
| Rule name | `SmartHomeTelemetryRule` |
| Description | `Forward smart home telemetry messages to SmartHomeIngest Lambda` |

Sau đó chọn **Next**.

### Bước 3: Nhập câu lệnh SQL

Chọn SQL version:

```text
2016-03-23
```

Nhập câu lệnh:

```sql
SELECT *, 'telemetry' AS recordType
FROM 'smarthome/home01/esp32-01/telemetry'
```

Ý nghĩa của câu lệnh:

| Thành phần | Ý nghĩa |
|---|---|
| `SELECT *` | Giữ lại toàn bộ thuộc tính trong payload |
| `'telemetry' AS recordType` | Bổ sung trường `recordType` với giá trị `telemetry` |
| `FROM '...'` | Chỉ nhận bản tin từ telemetry topic |

Trường `recordType` giúp `SmartHomeIngest` lựa chọn logic xử lý phù hợp mà không cần suy đoán loại dữ liệu từ nội dung payload.

Chọn **Next**.

### Bước 4: Chọn Lambda action

Trong phần **Rule actions**, chọn:

```text
Action 1
→ Lambda
```

Chọn Lambda function:

```text
SmartHomeIngest
```

Nếu giao diện hiển thị tùy chọn version hoặc alias, có thể để mặc định để gọi phiên bản hiện tại của hàm.

Chọn **Next**, kiểm tra lại cấu hình và chọn:

```text
Create
```

Khi tạo action bằng AWS Console, AWS thường tự thêm quyền để AWS IoT Core gọi Lambda. Nếu giao diện yêu cầu xác nhận cấp quyền, hãy chấp nhận yêu cầu này.

![Cấu hình SmartHomeTelemetryRule](/images/5.9.3.png)

> Ảnh minh chứng: chụp trang chi tiết rule, trong đó nhìn thấy tên rule, câu lệnh SQL và Lambda action `SmartHomeIngest`.

## 5.9.4. Tạo rule tiếp nhận events

Quay lại:

```text
AWS IoT Core
→ Message routing
→ Rules
→ Create rule
```

### Bước 1: Nhập thông tin rule

| Thuộc tính | Giá trị |
|---|---|
| Rule name | `SmartHomeEventsRule` |
| Description | `Forward smart home event messages to SmartHomeIngest Lambda` |

Chọn **Next**.

### Bước 2: Nhập câu lệnh SQL

Chọn SQL version:

```text
2016-03-23
```

Nhập:

```sql
SELECT *, 'event' AS recordType
FROM 'smarthome/home01/esp32-01/events'
```

Rule này chỉ được kích hoạt khi có bản tin được publish lên events topic. Payload gửi đến Lambda được bổ sung:

```json
{
  "recordType": "event"
}
```

Các trường khác trong payload ban đầu vẫn được giữ nguyên.

### Bước 3: Chọn Lambda action

Chọn:

```text
Action 1
→ Lambda
→ SmartHomeIngest
```

Sau đó chọn **Next** và **Create**.

![Cấu hình SmartHomeEventsRule](/images/5.9.4.png)

> Ảnh minh chứng: chụp trang chi tiết rule, trong đó nhìn thấy tên rule, câu lệnh SQL và Lambda action.

## 5.9.5. Kiểm tra trạng thái hai rule

Mở:

```text
AWS IoT Core
→ Message routing
→ Rules
```

Danh sách phải có hai rule:

| Rule | Trạng thái | Lambda action |
|---|---|---|
| `SmartHomeTelemetryRule` | Enabled | `SmartHomeIngest` |
| `SmartHomeEventsRule` | Enabled | `SmartHomeIngest` |

Nếu một rule đang ở trạng thái **Disabled**, chọn rule và bật lại trước khi kiểm thử.

![Hai AWS IoT Rule đã được bật](/images/5.9.5.png)



## 5.9.6. Kiểm tra quyền gọi Lambda

Việc gắn action vào rule chưa đủ nếu AWS IoT Core không có quyền gọi Lambda.

Mở:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Configuration
→ Permissions
→ Resource-based policy statements
```

Kiểm tra các permission statement dành cho AWS IoT Core. Nội dung cần thể hiện:

| Thuộc tính | Giá trị cần có |
|---|---|
| Principal | `iot.amazonaws.com` |
| Action | `lambda:InvokeFunction` |
| Source rule | `SmartHomeTelemetryRule` hoặc `SmartHomeEventsRule` |

Source ARN tương ứng có dạng:

```text
arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:rule/SmartHomeTelemetryRule
```

và:

```text
arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:rule/SmartHomeEventsRule
```

Thay `<ACCOUNT_ID>` bằng AWS account ID của bạn nhưng nên che một phần thông tin này khi đưa ảnh lên website công khai.

![Quyền AWS IoT gọi SmartHomeIngest](/images/5.9.6.png)



### Vì sao quyền này không nằm trong execution role?

Hai loại quyền có mục đích khác nhau:

- `SmartHomeIngestRole` cho phép Lambda truy cập các dịch vụ khác như DynamoDB, SNS và CloudWatch.
- Resource-based policy của Lambda cho phép AWS IoT Core gọi `SmartHomeIngest`.

Nếu thiếu permission statement, hãy mở lại IoT Rule, chỉnh sửa Lambda action và chọn lại `SmartHomeIngest`. AWS Console sẽ yêu cầu hoặc tự động thêm quyền cần thiết. Không nên cấp quyền gọi Lambda cho mọi nguồn bằng wildcard nếu không cần thiết.

## 5.9.7. Kiểm tra nhanh luồng dữ liệu

Vì hệ thống đã gửi dữ liệu đến AWS IoT Core, không cần cấu hình lại thiết bị hoặc Home Assistant.

Sau khi hai rule được bật:

1. Chờ một bản tin telemetry mới.
2. Mở Lambda `SmartHomeIngest`.
3. Kiểm tra chỉ số **Invocations** trong tab **Monitor**.
4. Mở bảng DynamoDB `SmartHomeTelemetry`.
5. Kiểm tra item mới có `recordType` bằng `telemetry`.
6. Khi có sự kiện, kiểm tra item tương ứng có `recordType` bằng `event`.

Phần kiểm thử đầy đủ bằng MQTT Test Client, DynamoDB, SNS và CloudWatch sẽ được trình bày ở mục 5.12.

## 5.9.8. Một số lỗi thường gặp

### Rule không được kích hoạt

Kiểm tra:

- Rule đang ở trạng thái `Enabled`.
- Region đang là `ap-southeast-1`.
- Tên topic trong câu lệnh SQL khớp hoàn toàn với topic thực tế.
- Không có khoảng trắng hoặc sai chữ hoa, chữ thường trong topic.

### Lambda không có invocation mới

Kiểm tra:

- Rule đã có Lambda action `SmartHomeIngest`.
- Resource-based policy có principal `iot.amazonaws.com`.
- Source ARN trỏ đến đúng tên rule.
- Lambda và IoT Rule nằm trong cùng Region.

### Lambda được gọi nhưng không có dữ liệu trong DynamoDB

Mở CloudWatch Logs của `SmartHomeIngest` và kiểm tra:

- Payload có đúng định dạng JSON.
- `recordType` có giá trị `telemetry` hoặc `event`.
- Execution role có quyền ghi vào bảng `SmartHomeTelemetry`.
- Không xuất hiện `AccessDeniedException` hoặc lỗi thiếu trường dữ liệu.

## 5.9.9. Kết quả đạt được

Sau khi hoàn thành phần này:

- Telemetry topic đã được liên kết với `SmartHomeTelemetryRule`.
- Events topic đã được liên kết với `SmartHomeEventsRule`.
- Hai rule đều gọi Lambda `SmartHomeIngest`.
- Payload được bổ sung trường `recordType`.
- AWS IoT Core đã được cấp quyền gọi Lambda.
- Dữ liệu MQTT có thể tiếp tục đi vào luồng xử lý và lưu trữ trên AWS.

## Tài liệu tham khảo

- [AWS IoT Rules](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html)
- [AWS IoT SQL reference](https://docs.aws.amazon.com/iot/latest/developerguide/iot-sql-reference.html)
- [AWS IoT Lambda rule action](https://docs.aws.amazon.com/iot/latest/developerguide/lambda-rule-action.html)