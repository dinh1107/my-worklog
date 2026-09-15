---
title: "Cấu hình cảnh báo Amazon SNS"
date: 2026-09-14
weight: 10
chapter: false
pre: "<b>5.10. </b>"
---

# Cấu hình Amazon SNS gửi email cảnh báo

Trong phần trước, AWS IoT Rules đã chuyển telemetry và events đến Lambda `SmartHomeIngest`.

Dữ liệu telemetry chỉ cần được lưu định kỳ. Tuy nhiên, các sự kiện an toàn cần được thông báo cho người dùng ngay khi xảy ra. Trong phần này, chúng ta sử dụng Amazon Simple Notification Service — Amazon SNS — để gửi cảnh báo qua email.

Hệ thống hỗ trợ ba loại thông báo:

| Event type | Ý nghĩa |
|---|---|
| `gas_alarm_started` | Phát hiện mức gas vượt ngưỡng |
| `gas_alarm_cleared` | Mức gas đã trở lại an toàn |
| `door_brute_force_detected` | Nhập sai mật khẩu cửa ba lần |

## 5.10.1. Vai trò của Amazon SNS

Amazon SNS là dịch vụ gửi thông báo theo mô hình publish/subscribe.

Trong workshop này:

- Lambda `SmartHomeIngest` là bên phát thông báo.
- SNS topic `SmartHomeAlerts` là kênh phân phối.
- Địa chỉ email là endpoint nhận thông báo.

Luồng cảnh báo được thực hiện như sau:

```text
Sự kiện MQTT
→ AWS IoT Rule
→ SmartHomeIngest
→ Amazon SNS
→ Email người dùng
```

Việc sử dụng SNS giúp tách logic xử lý dữ liệu khỏi phương thức nhận cảnh báo. Sau này, hệ thống có thể bổ sung nhiều người nhận hoặc endpoint khác mà không phải thay đổi thiết bị IoT.

## 5.10.2. Thông tin cấu hình

Kiểm tra Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Sử dụng các giá trị sau:

| Thành phần | Giá trị |
|---|---|
| Topic type | `Standard` |
| Topic name | `SmartHomeAlerts` |
| Display name | `SmartHome Alerts` |
| Subscription protocol | `Email` |
| Lambda function | `SmartHomeIngest` |
| Lambda execution role | `SmartHomeIngestRole` |
| Environment variable | `SNS_TOPIC_ARN` |

Topic loại **Standard** phù hợp vì hệ thống cần gửi cảnh báo nhanh và không yêu cầu thứ tự nghiêm ngặt giữa các email.

## 5.10.3. Tạo SNS topic

Thực hiện bước này bằng tài khoản root hoặc tài khoản quản trị nếu IAM user chưa có quyền tạo SNS topic.

### Bước 1: Mở Amazon SNS

Trong AWS Management Console, tìm kiếm:

```text
Amazon Simple Notification Service
```

Mở:

```text
Amazon SNS
→ Topics
→ Create topic
```

### Bước 2: Nhập thông tin topic

Trong phần **Details**, nhập:

| Thuộc tính | Giá trị |
|---|---|
| Type | `Standard` |
| Name | `SmartHomeAlerts` |
| Display name | `SmartHome Alerts` |

Các phần còn lại có thể giữ mặc định:

- Không cần bật FIFO.
- Không cần cấu hình delivery retry cho email.
- Không cần tạo customer managed KMS key.
- Giữ access policy mặc định.

Chọn:

```text
Create topic
```

Sau khi tạo thành công, AWS hiển thị topic ARN có dạng:

```text
arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts
```

Sao chép ARN này để cấu hình cho Lambda ở bước sau.

![Thông tin topic SmartHomeAlerts](/images/5.10.3.png)

> Ảnh minh chứng: chụp tên topic, loại Standard và ARN. Nên che một phần AWS account ID trước khi đưa ảnh lên website công khai.

## 5.10.4. Tạo email subscription

Trong trang chi tiết topic `SmartHomeAlerts`, chọn:

```text
Create subscription
```

Điền các thông tin:

| Thuộc tính | Giá trị |
|---|---|
| Topic ARN | ARN của `SmartHomeAlerts` |
| Protocol | `Email` |
| Endpoint | Địa chỉ email nhận cảnh báo |

Chọn:

```text
Create subscription
```

Subscription ban đầu sẽ có trạng thái:

```text
Pending confirmation
```

### Xác nhận địa chỉ email

Mở hộp thư của địa chỉ vừa đăng ký và tìm email có tiêu đề tương tự:

```text
AWS Notification - Subscription Confirmation
```

Mở email và chọn:

```text
Confirm subscription
```

Quay lại Amazon SNS và làm mới trang. Trạng thái phải chuyển thành:

```text
Confirmed
```

Nếu không thấy email:

- Kiểm tra thư mục Spam hoặc Junk.
- Kiểm tra lại địa chỉ email đã nhập.
- Chờ vài phút rồi gửi lại confirmation nếu cần.
- Không tạo nhiều subscription cho cùng một địa chỉ.

![Email subscription đã được xác nhận](/images/5.10.4.png)



## 5.10.5. Cấp quyền publish cho Lambda

Lambda chỉ có thể gửi thông báo khi execution role có quyền `sns:Publish`.

Thực hiện bước này bằng root hoặc tài khoản quản trị.

Mở:

```text
IAM
→ Roles
→ SmartHomeIngestRole
→ Permissions
→ Add permissions
→ Create inline policy
→ JSON
```

Nhập policy sau:

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

Thay `<ACCOUNT_ID>` bằng AWS account ID thực tế.

Chọn **Next** và đặt tên policy:

```text
SmartHomeIngestSNSPublish
```

Sau đó chọn:

```text
Create policy
```

Policy chỉ cho phép Lambda publish đến `SmartHomeAlerts`, thay vì cho phép truy cập tất cả SNS topic trong tài khoản. Đây là nguyên tắc cấp quyền tối thiểu.

## 5.10.6. Khai báo SNS topic cho Lambda

Đăng xuất tài khoản root hoặc quản trị, sau đó đăng nhập lại bằng IAM user `dinh-fcj`.

Mở:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Configuration
→ Environment variables
→ Edit
```

Thêm biến môi trường:

| Key | Value |
|---|---|
| `SNS_TOPIC_ARN` | ARN của topic `SmartHomeAlerts` |

Ví dụ:

```text
SNS_TOPIC_ARN
arn:aws:sns:ap-southeast-1:<ACCOUNT_ID>:SmartHomeAlerts
```

Chọn:

```text
Save
```

Tên biến phải giống chính xác tên được sử dụng trong mã nguồn của `SmartHomeIngest`.

Việc sử dụng biến môi trường giúp không phải ghi trực tiếp topic ARN vào mã nguồn. Khi thay đổi SNS topic, chúng ta chỉ cần cập nhật cấu hình Lambda.

## 5.10.7. Logic gửi cảnh báo trong Lambda

Trong mã nguồn `SmartHomeIngest`, SNS client được khởi tạo bằng AWS SDK for Python:

```python
import boto3
import os

sns = boto3.client("sns")
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]
```

Khi nhận một event cần thông báo, Lambda gọi:

```python
sns.publish(
    TopicArn=SNS_TOPIC_ARN,
    Subject=subject,
    Message=message
)
```

Lambda chỉ gửi email đối với dữ liệu có:

```json
{
  "recordType": "event"
}
```

Ví dụ cách ánh xạ sự kiện:

| Event type | Tiêu đề email |
|---|---|
| `gas_alarm_started` | `CẢNH BÁO: Phát hiện rò rỉ gas` |
| `gas_alarm_cleared` | `THÔNG BÁO: Mức gas đã an toàn` |
| `door_brute_force_detected` | `CẢNH BÁO: Nhập sai mật khẩu cửa` |

Telemetry gửi mỗi 30 giây không tạo email. Điều này giúp tránh gửi quá nhiều thông báo không cần thiết.

## 5.10.8. Kiểm tra topic độc lập

Trước khi kiểm tra toàn bộ hệ thống, có thể xác nhận SNS topic hoạt động độc lập.

Mở:

```text
Amazon SNS
→ Topics
→ SmartHomeAlerts
→ Publish message
```

Nhập:

| Thuộc tính | Giá trị |
|---|---|
| Subject | `Smart Home SNS Test` |
| Message body | `Amazon SNS email notification is working.` |

Chọn:

```text
Publish message
```

Kiểm tra hộp thư đã đăng ký. Nếu nhận được email, topic và subscription đang hoạt động đúng.

## 5.10.9. Kiểm tra cảnh báo thực tế

Khi hệ thống phát sinh một trong ba sự kiện, email sẽ được gửi đến địa chỉ đã xác nhận.

Ví dụ đối với sự kiện gas:

```json
{
  "eventType": "gas_alarm_started",
  "gasRaw": 2000,
  "threshold": 1500
}
```

Kết quả mong đợi:

- Lambda `SmartHomeIngest` được gọi.
- Sự kiện được lưu vào DynamoDB.
- SNS gửi email cảnh báo.
- Email chứa tên thiết bị, loại sự kiện và thời gian xảy ra.

![Email cảnh báo từ hệ thống](/images/5.10.9.png)



## 5.10.10. Một số lỗi thường gặp

### Subscription vẫn ở trạng thái Pending confirmation

Kiểm tra:

- Hộp thư Spam hoặc Junk.
- Địa chỉ email trong endpoint.
- Liên kết **Confirm subscription** đã được mở.
- Trang SNS đã được làm mới sau khi xác nhận.

SNS sẽ không gửi thông báo đến email chưa được xác nhận.

### Lambda báo lỗi AccessDeniedException

Nếu CloudWatch hiển thị lỗi liên quan đến `sns:Publish`, kiểm tra:

- Execution role là `SmartHomeIngestRole`.
- Role có policy `SmartHomeIngestSnsPublishPolicy`.
- Resource ARN trong policy khớp với `SmartHomeAlerts`.
- Lambda và SNS topic nằm trong cùng Region.

### Lambda chạy thành công nhưng không có email

Kiểm tra:

- Subscription đang ở trạng thái `Confirmed`.
- `SNS_TOPIC_ARN` có đúng topic ARN hay không.
- Payload có `recordType` bằng `event`.
- `eventType` có nằm trong danh sách sự kiện được Lambda xử lý.
- Email có nằm trong Spam hoặc Promotions hay không.

### Telemetry không gửi email

Đây là hành vi đúng. Telemetry được gửi định kỳ để lưu trữ và không phải sự kiện cảnh báo.

## 5.10.11. Kết quả đạt được

Sau khi hoàn thành phần này:

- SNS topic `SmartHomeAlerts` đã được tạo.
- Địa chỉ email đã đăng ký và xác nhận subscription.
- `SmartHomeIngestRole` có quyền `sns:Publish`.
- Lambda nhận topic ARN thông qua biến `SNS_TOPIC_ARN`.
- Các sự kiện gas và an ninh có thể tạo email cảnh báo.
- Telemetry định kỳ không làm phát sinh email không cần thiết.

## Tài liệu tham khảo

- [Creating an Amazon SNS topic](https://docs.aws.amazon.com/sns/latest/dg/sns-create-topic.html)
- [Amazon SNS email subscription setup](https://docs.aws.amazon.com/sns/latest/dg/sns-email-notifications.html)
- [AWS Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)
- [AWS Lambda execution roles](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)