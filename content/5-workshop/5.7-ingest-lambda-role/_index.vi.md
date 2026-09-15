---
title: "Tạo IAM execution role cho Lambda xử lý dữ liệu"
weight: 7
chapter: false
pre: "<b>5.7. </b>"
---

# Tạo IAM execution role cho Lambda xử lý dữ liệu

Trong phần này, chúng ta tạo IAM execution role cho Lambda `SmartHomeIngest`. Role cho phép Lambda ghi nhật ký vào Amazon CloudWatch Logs và ghi dữ liệu vào bảng DynamoDB `SmartHomeTelemetry`.

Phần này được thực hiện bằng tài khoản root hoặc tài khoản quản trị vì IAM user `dinh-fcj` không có quyền tạo role và quản lý policy IAM.

## 5.7.1. Mục tiêu và phân tách quyền

Sau khi hoàn thành, hệ thống có các thành phần sau:

| Thành phần | Tên | Được gắn vào |
|---|---|---|
| Lambda execution role | `SmartHomeIngestRole` | Lambda `SmartHomeIngest` |
| AWS managed policy | `AWSLambdaBasicExecutionRole` | `SmartHomeIngestRole` |
| Inline policy | `SmartHomeIngestDynamoDBWrite` | `SmartHomeIngestRole` |
| Developer policy | `SmartHomeLambdaDeveloperPolicy` | Group `SmartHome-Developers` |

Hai loại quyền có vai trò khác nhau:

- **Execution role** là quyền mà Lambda sử dụng trong lúc chạy.
- **Developer policy** là quyền của IAM user để tạo Lambda và truyền execution role cho Lambda.

Không gắn `SmartHomeLambdaDeveloperPolicy` vào execution role và không gắn `SmartHomeIngestDynamoDBWrite` vào IAM user.

## 5.7.2. Tạo SmartHomeIngestRole

Đăng nhập bằng tài khoản root hoặc tài khoản quản trị, sau đó mở:

```text
IAM → Roles → Create role
```

Tại **Trusted entity type**, chọn:

```text
AWS service
```

Tại **Service or use case**, chọn:

```text
Lambda
```

Chọn **Next** để chuyển sang bước cấp quyền.

Việc chọn Lambda làm trusted service cho phép dịch vụ `lambda.amazonaws.com` sử dụng role thông qua hành động `sts:AssumeRole`.

Trust policy được AWS tạo có dạng:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

![Chọn AWS Lambda làm trusted service](/images/5.7.2.png)

## 5.7.3. Gắn quyền ghi CloudWatch Logs

Trong danh sách permission policies, tìm và chọn:

```text
AWSLambdaBasicExecutionRole
```

Policy do AWS quản lý này cho phép Lambda:

- Tạo CloudWatch log group.
- Tạo log stream.
- Ghi các sự kiện log trong quá trình thực thi.

Chọn **Next**, sau đó nhập tên role:

```text
SmartHomeIngestRole
```

Kiểm tra trusted entity là Lambda và permission policy là `AWSLambdaBasicExecutionRole`, rồi chọn:

```text
Create role
```

![Tạo role SmartHomeIngestRole](/images/5.7.3.png)

## 5.7.4. Cấp quyền ghi dữ liệu vào DynamoDB

Role vừa tạo mới chỉ có quyền ghi CloudWatch Logs. Lambda cần thêm quyền ghi item vào bảng `SmartHomeTelemetry`.

Mở:

```text
IAM → Roles → SmartHomeIngestRole
→ Permissions → Add permissions
→ Create inline policy
```

Chọn tab **JSON** và nhập:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteSmartHomeTelemetry",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:ap-southeast-1:079755087993:table/SmartHomeTelemetry"
    }
  ]
}
```

Chọn **Next** và đặt tên policy:

```text
SmartHomeIngestDynamoDBWrite
```

Sau đó chọn **Create policy**.

Policy chỉ cho phép `dynamodb:PutItem` trên đúng bảng `SmartHomeTelemetry`. Lambda không được cấp `AmazonDynamoDBFullAccess` vì không cần quản trị các bảng DynamoDB khác.

![Inline policy SmartHomeIngestDynamoDBWrite](/images/5.7.3.png)



## 5.7.5. Kiểm tra quyền triển khai của IAM user

Lambda `SmartHomeIngest` sẽ được tạo bằng IAM user `dinh-fcj`. Trong quá trình thực hành, root đã tạo policy:

```text
SmartHomeLambdaDeveloperPolicy
```

và gắn policy này vào group:

```text
SmartHome-Developers
```

Policy này bao gồm:

- Quyền tạo, xem, cập nhật và chạy thử Lambda `SmartHomeIngest`.
- Quyền xem `SmartHomeIngestRole` trong giao diện Lambda.
- Quyền `iam:PassRole` đối với `SmartHomeIngestRole`.
- Điều kiện chỉ cho phép truyền role đến `lambda.amazonaws.com`.

Mở:

```text
IAM → User groups → SmartHome-Developers → Permissions
```

Xác nhận `SmartHomeLambdaDeveloperPolicy` đã được gắn vào group.

![SmartHomeLambdaDeveloperPolicy được gắn vào group](/images/5.7.5.png)


## 5.7.6. Kiểm tra kết quả

Mở:

```text
IAM → Roles → SmartHomeIngestRole
```

Kiểm tra các giá trị:

| Nội dung | Kết quả cần có |
|---|---|
| Trusted service | `lambda.amazonaws.com` |
| Managed policy | `AWSLambdaBasicExecutionRole` |
| Inline policy | `SmartHomeIngestDynamoDBWrite` |
| DynamoDB action | `dynamodb:PutItem` |
| DynamoDB resource | Bảng `SmartHomeTelemetry` |
| SNS permission | Chưa được thêm ở bước này |

Sau đó kiểm tra group `SmartHome-Developers` đã có `SmartHomeLambdaDeveloperPolicy`.

Khi mọi giá trị đều chính xác, đăng xuất tài khoản root và đăng nhập lại bằng IAM user:

```text
dinh-fcj
```

IAM user này sẽ sử dụng `SmartHomeIngestRole` khi tạo Lambda `SmartHomeIngest` trong phần tiếp theo.
