---
title: "Tự động xuất báo cáo"
date: 2026-09-14
weight: 15
chapter: false
pre: "<b>5.15. </b>"
---

# Tự động xuất báo cáo bằng EventBridge Scheduler

Trong phần trước, Lambda `SmartHomeExport` đã có thể tạo báo cáo JSON và CSV khi được chạy thủ công.

Trong phần này, chúng ta sử dụng Amazon EventBridge Scheduler để tự động gọi Lambda vào:

```text
00:05 mỗi ngày theo giờ Việt Nam
```

Khi nhận payload rỗng `{}`, Lambda sẽ tự chọn ngày hôm trước và xuất báo cáo vào Amazon S3.

Luồng tự động:

```text
00:05 Asia/Ho_Chi_Minh
→ EventBridge Scheduler
→ SmartHomeExport
→ DynamoDB
→ report.json và report.csv
→ Amazon S3
```

## 5.15.1. Vai trò của EventBridge Scheduler

EventBridge Scheduler cho phép gọi một dịch vụ AWS theo lịch:

- Một lần tại thời điểm xác định.
- Theo khoảng thời gian cố định.
- Theo biểu thức cron.
- Theo một múi giờ cụ thể.

Project sử dụng cron-based schedule vì báo cáo cần được tạo vào một thời điểm cố định mỗi ngày.

Việc chạy lúc `00:05` thay vì đúng `00:00` tạo ra khoảng đệm 5 phút để dữ liệu cuối ngày trước được gửi và lưu vào DynamoDB.

## 5.15.2. Thông tin cấu hình

| Thành phần | Giá trị |
|---|---|
| Schedule name | `SmartHomeDailyExport` |
| Schedule group | `default` |
| Schedule type | Recurring |
| Expression | `cron(5 0 * * ? *)` |
| Time zone | `Asia/Ho_Chi_Minh` |
| Flexible time window | Off |
| Target | `SmartHomeExport` |
| Payload | `{}` |
| Execution role | `SmartHomeSchedulerRole` |
| Maximum event age | 1 giờ |
| Maximum retries | 2 |
| Dead-letter queue | None |
| State | Enabled |

## 5.15.3. Phân biệt hai execution role

Project sử dụng hai role khác nhau:

| Role | Được sử dụng bởi | Mục đích |
|---|---|---|
| `SmartHomeExportRole` | Lambda | Cho phép đọc DynamoDB, ghi S3 và ghi CloudWatch Logs |
| `SmartHomeSchedulerRole` | EventBridge Scheduler | Cho phép Scheduler gọi `SmartHomeExport` |

Scheduler không sử dụng `SmartHomeExportRole` để gọi Lambda. Mỗi dịch vụ cần một role đúng với trách nhiệm của nó.

## 5.15.4. Tạo execution role cho Scheduler

Thực hiện bước này bằng tài khoản root hoặc tài khoản quản trị.

Mở:

```text
IAM
→ Roles
→ Create role
→ Custom trust policy
```

Nhập trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "scheduler.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "<ACCOUNT_ID>",
          "aws:SourceArn": "arn:aws:scheduler:ap-southeast-1:<ACCOUNT_ID>:schedule-group/default"
        }
      }
    }
  ]
}
```

Thay `<ACCOUNT_ID>` bằng AWS account ID thực tế.

Điều kiện `aws:SourceAccount` và `aws:SourceArn` giới hạn việc sử dụng role trong đúng tài khoản và schedule group `default`.

Chọn **Next**. Chưa cần chọn managed policy.

Đặt tên:

```text
SmartHomeSchedulerRole
```

Sau đó chọn:

```text
Create role
```

## 5.15.5. Cho phép Scheduler gọi Lambda

Mở:

```text
IAM
→ Roles
→ SmartHomeSchedulerRole
→ Permissions
→ Add permissions
→ Create inline policy
→ JSON
```

Nhập:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeSmartHomeExport",
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": [
        "arn:aws:lambda:ap-southeast-1:<ACCOUNT_ID>:function:SmartHomeExport",
        "arn:aws:lambda:ap-southeast-1:<ACCOUNT_ID>:function:SmartHomeExport:*"
      ]
    }
  ]
}
```

Đặt tên policy:

```text
SmartHomeSchedulerInvokeLambda
```

Chọn:

```text
Create policy
```

Policy này chỉ cho phép Scheduler gọi `SmartHomeExport`. Nó không cho phép gọi tất cả Lambda function trong tài khoản.

## 5.15.6. Cấp quyền Scheduler cho IAM user

IAM user `dinh-fcj` cần quyền tạo schedule và truyền `SmartHomeSchedulerRole` cho Scheduler.

Vẫn sử dụng tài khoản root hoặc quản trị, mở:

```text
IAM
→ Policies
→ Create policy
→ JSON
```

Nhập:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ViewSchedulerConsole",
      "Effect": "Allow",
      "Action": [
        "scheduler:ListSchedules",
        "scheduler:ListScheduleGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ViewDefaultScheduleGroup",
      "Effect": "Allow",
      "Action": [
        "scheduler:GetScheduleGroup"
      ],
      "Resource": "arn:aws:scheduler:ap-southeast-1:<ACCOUNT_ID>:schedule-group/default"
    },
    {
      "Sid": "ManageSmartHomeDailyExport",
      "Effect": "Allow",
      "Action": [
        "scheduler:CreateSchedule",
        "scheduler:GetSchedule",
        "scheduler:UpdateSchedule",
        "scheduler:ListTagsForResource",
        "scheduler:TagResource",
        "scheduler:UntagResource"
      ],
      "Resource": "arn:aws:scheduler:ap-southeast-1:<ACCOUNT_ID>:schedule/default/SmartHomeDailyExport"
    },
    {
      "Sid": "PassOnlySchedulerRole",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/SmartHomeSchedulerRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "scheduler.amazonaws.com"
        }
      }
    },
    {
      "Sid": "ViewSchedulerRole",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:ListRoles"
      ],
      "Resource": "*"
    }
  ]
}
```

Đặt tên:

```text
SmartHomeSchedulerDeveloperPolicy
```

Chọn:

```text
Create policy
```

Gắn policy vào nhóm:

```text
IAM
→ User groups
→ SmartHome-Developers
→ Permissions
→ Add permissions
→ Attach policies
```

Chọn:

```text
SmartHomeSchedulerDeveloperPolicy
```

Sau đó đăng xuất tài khoản root và đăng nhập lại bằng:

```text
dinh-fcj
```

## 5.15.7. Tạo lịch xuất báo cáo

Kiểm tra Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Mở:

```text
Amazon EventBridge
→ Scheduler
→ Schedules
→ Create schedule
```

### Nhập thông tin schedule

| Thuộc tính | Giá trị |
|---|---|
| Schedule name | `SmartHomeDailyExport` |
| Schedule group | `default` |
| Description | `Export previous day smart home report to S3` |
| Occurrence | `Recurring schedule` |

### Chọn lịch dạng cron

Chọn:

```text
Cron-based schedule
```

Nhập biểu thức:

```text
cron(5 0 * * ? *)
```

Nếu giao diện chia biểu thức thành sáu ô, nhập:

| Trường | Giá trị |
|---|---|
| Minutes | `5` |
| Hours | `0` |
| Day of month | `*` |
| Month | `*` |
| Day of week | `?` |
| Year | `*` |

Biểu thức này có nghĩa:

```text
Chạy mỗi ngày lúc 00:05
```

### Chọn múi giờ

Chọn:

```text
Asia/Ho_Chi_Minh
```

Không chọn UTC. Nếu sử dụng `00:05 UTC`, schedule sẽ chạy lúc `07:05` theo giờ Việt Nam.

### Flexible time window

Chọn:

```text
Off
```

Khi Flexible time window tắt, Scheduler gọi target trong khoảng:

```text
00:05:00–00:05:59
```

### Timeframe

Để trống:

```text
Start date
End date
```

Schedule sẽ hoạt động sau khi được tạo và tiếp tục chạy không giới hạn.

Chọn:

```text
Next
```

## 5.15.8. Chọn Lambda làm target

Tại trang **Select target**, chọn:

```text
Templated targets
→ AWS Lambda
→ Invoke
```

Chọn function:

```text
SmartHomeExport
```

Nếu có lựa chọn version hoặc alias, có thể để trống để gọi phiên bản hiện hành.

Trong phần **Input**, nhập chính xác:

```json
{}
```

Không nhập `reportDate`.

Khi payload là `{}`, `SmartHomeExport` tự tính ngày hôm trước theo múi giờ `Asia/Ho_Chi_Minh`.

Chọn:

```text
Next
```

## 5.15.9. Cấu hình retry, mã hóa và quyền

### Schedule state

Chọn:

```text
Enable schedule
```

### Action after schedule completion

Chọn:

```text
None
```

Đây là recurring schedule nên không cần tự động xóa.

### Retry policy

Bật retry và đặt:

| Thuộc tính | Giá trị |
|---|---|
| Maximum age of event | `1 hour` hoặc `3600 seconds` |
| Maximum retry attempts | `2` |

Nếu Lambda tạm thời không thể được gọi, Scheduler sẽ thử lại tối đa hai lần trong giới hạn một giờ.

### Dead-letter queue

Chọn:

```text
None
```

Workshop chưa sử dụng SQS DLQ. Trong môi trường production, nên bổ sung DLQ để lưu các lần gọi thất bại sau khi hết retry.

### Encryption

Không chọn:

```text
Customize encryption settings (advanced)
```

Khi bỏ chọn tùy chỉnh mã hóa, Scheduler sử dụng cơ chế mã hóa mặc định do AWS quản lý.

Không cần:

- Tạo customer managed KMS key.
- Chọn một KMS alias.
- Cấp quyền `kms:ListAliases`.
- Thêm quyền KMS cho IAM user.

### Permissions

Chọn:

```text
Use existing role
```

Sau đó chọn:

```text
SmartHomeSchedulerRole
```

Không chọn **Create new role**, vì IAM user chỉ được phép truyền role đã được quản trị viên chuẩn bị.

Chọn:

```text
Next
```

![Target và execution role của schedule](/images/5.15.9.png)


## 5.15.10. Kiểm tra và tạo schedule

Tại trang **Review and create schedule**, xác nhận:

| Thuộc tính | Giá trị |
|---|---|
| Name | `SmartHomeDailyExport` |
| Group | `default` |
| State | Enabled |
| Expression | `cron(5 0 * * ? *)` |
| Time zone | `Asia/Ho_Chi_Minh` |
| Flexible window | Off |
| Target | `SmartHomeExport` |
| Input | `{}` |
| Execution role | `SmartHomeSchedulerRole` |
| Retry attempts | 2 |
| Maximum event age | 1 hour |

Chọn:

```text
Create schedule
```

Sau khi tạo thành công, mở schedule và kiểm tra:

```text
State: Enabled
```

Giá trị **Next invocation time** phải tương ứng với `00:05` của ngày tiếp theo theo giờ Việt Nam.

![Chi tiết SmartHomeDailyExport](/images/5.15.10.png)



## 5.15.11. Cách Lambda chọn ngày báo cáo

Giả sử Scheduler chạy vào:

```text
2026-09-15 00:05 Asia/Ho_Chi_Minh
```

Scheduler gửi:

```json
{}
```

Lambda tính ngày hôm trước:

```text
2026-09-14
```

Sau đó tạo:

```text
reports/home01/esp32-01/2026/09/14/report.json
reports/home01/esp32-01/2026/09/14/report.csv
```

Việc xác định ngày được thực hiện bên trong Lambda, không phải trong Scheduler. Nhờ vậy, không cần sửa payload mỗi ngày.

## 5.15.12. Kiểm tra lần chạy tự động

Sau thời điểm `00:05`, chờ khoảng 1–3 phút rồi mở:

```text
Amazon S3
→ smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
→ reports
→ home01
→ esp32-01
→ YYYY
→ MM
→ DD
```

Kiểm tra:

- Có `report.json`.
- Có `report.csv`.
- `Last modified` sau thời điểm Scheduler chạy.
- Kích thước file lớn hơn `0 B`.

![Báo cáo được Scheduler tạo tự động](/images/5.15.12.png)



Tiếp tục kiểm tra:

```text
AWS Lambda
→ SmartHomeExport
→ Monitor
```

Kết quả mong đợi:

| Chỉ số | Kết quả |
|---|---|
| Invocations | Tăng thêm 1 |
| Errors | 0 |
| Throttles | 0 |
| Duration | Nhỏ hơn timeout 30 giây |

CloudWatch Metrics có thể cập nhật chậm vài phút.

## 5.15.13. Ảnh hưởng của S3 Versioning

Nếu báo cáo của ngày hôm trước đã tồn tại từ lần Lambda Test, Scheduler sẽ ghi lại cùng object key.

Do S3 Versioning đã bật:

- File cũ không bị mất hoàn toàn.
- Báo cáo do Scheduler tạo trở thành current version.
- Báo cáo từ lần chạy thủ công trở thành noncurrent version.

Có thể bật:

```text
Show versions
```

để kiểm tra các phiên bản.



## 5.15.14. Một số lỗi thường gặp

### Không chọn được SmartHomeSchedulerRole

Kiểm tra:

- Role đã được tạo.
- Trust policy cho phép `scheduler.amazonaws.com`.
- IAM user có `iam:GetRole`.
- IAM user có `iam:PassRole`.
- Điều kiện `iam:PassedToService` là `scheduler.amazonaws.com`.

### Xuất hiện lỗi kms:ListAliases

Bỏ chọn:

```text
Customize encryption settings (advanced)
```

Workshop không sử dụng customer managed KMS key nên không cần thêm quyền KMS.

### Schedule Enabled nhưng Lambda không được gọi

Kiểm tra:

- Target là `SmartHomeExport`.
- Execution role là `SmartHomeSchedulerRole`.
- Role có `lambda:InvokeFunction`.
- ARN Lambda trong policy đúng Region và account.
- `Next invocation time` đã thực sự đến.

### Schedule chạy sai giờ

Kiểm tra time zone:

```text
Asia/Ho_Chi_Minh
```

Không sử dụng UTC cho biểu thức trong workshop này.

### Lambda được gọi nhưng S3 không có báo cáo

Mở CloudWatch Logs của:

```text
/aws/lambda/SmartHomeExport
```

Tìm:

```text
AccessDeniedException
ResourceNotFoundException
Task timed out
```

Đồng thời kiểm tra các biến:

```text
TABLE_NAME
REPORT_BUCKET
DEVICE_KEY
HOME_ID
DEVICE_ID
```

### Báo cáo có 0 bản ghi

Điều này có thể xảy ra nếu ngày hôm trước không có dữ liệu. Schedule và Lambda vẫn có thể hoạt động thành công.

Kiểm tra `reportDate`, `DEVICE_KEY` và timestamp của các item DynamoDB.

## 5.15.15. Kết quả đạt được

Sau khi hoàn thành phần này:

- `SmartHomeSchedulerRole` đã được tạo.
- Role chỉ có quyền gọi `SmartHomeExport`.
- IAM user có quyền quản lý schedule cụ thể và truyền đúng role.
- `SmartHomeDailyExport` chạy lúc 00:05 giờ Việt Nam.
- Scheduler gửi payload `{}` đến Lambda.
- Lambda tự xuất dữ liệu của ngày hôm trước.
- Báo cáo JSON và CSV được lưu vào S3.
- S3 Versioning giữ lại các phiên bản báo cáo.
- Luồng xuất báo cáo hằng ngày hoạt động tự động.

## Tài liệu tham khảo

- [Getting started with EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/getting-started.html)
- [Schedule types in EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html)
- [Using templated targets](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-templated.html)
- [Confused deputy prevention](https://docs.aws.amazon.com/scheduler/latest/UserGuide/cross-service-confused-deputy-prevention.html)