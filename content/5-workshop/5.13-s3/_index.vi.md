---
title: "Tạo S3 bucket lưu báo cáo"
date: 2026-09-14
weight: 13
chapter: false
pre: "<b>5.13. </b>"
---

# Tạo Amazon S3 bucket lưu báo cáo

DynamoDB đang lưu từng bản ghi telemetry và event của hệ thống. Tuy nhiên, DynamoDB không phải nơi phù hợp để lưu các file báo cáo hoàn chỉnh phục vụ tải xuống hoặc chia sẻ.

Trong phần này, chúng ta sẽ tạo một Amazon S3 bucket để lưu báo cáo hằng ngày dưới hai định dạng:

```text
JSON
CSV
```

Lambda xuất báo cáo sẽ được tạo ở phần tiếp theo.

## 5.13.1. Vai trò của Amazon S3

Amazon Simple Storage Service — Amazon S3 — là dịch vụ lưu trữ object. Mỗi file được lưu dưới dạng một object bên trong bucket.

Trong project này:

| Dịch vụ | Dữ liệu được lưu |
|---|---|
| DynamoDB | Các bản ghi telemetry và event riêng lẻ |
| Amazon S3 | Các file báo cáo tổng hợp JSON và CSV |

Luồng xuất báo cáo dự kiến:

```text
DynamoDB
→ SmartHomeExport
→ Amazon S3
```

S3 được sử dụng vì:

- Phù hợp để lưu file.
- Hỗ trợ tải báo cáo xuống.
- Có khả năng lưu nhiều phiên bản của cùng một file.
- Dữ liệu được mã hóa mặc định.
- Có thể kiểm soát quyền truy cập bằng IAM.
- Không cần duy trì máy chủ lưu trữ riêng.

## 5.13.2. Quy ước đặt tên

S3 bucket name phải duy nhất trong AWS partition. Vì vậy, chúng ta bổ sung AWS account ID và Region vào tên bucket.

Sử dụng quy ước:

```text
smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

Ví dụ:

```text
smarthome-reports-123456789012-ap-southeast-1
```

Thay `<ACCOUNT_ID>` bằng AWS account ID của bạn.

Không nên sử dụng email, họ tên, mật khẩu hoặc thông tin nhạy cảm trong bucket name vì tên bucket có thể xuất hiện trong ARN và URL của object.

## 5.13.3. Thông tin cấu hình

| Thuộc tính | Giá trị |
|---|---|
| Bucket type | General purpose |
| Bucket name | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |
| Region | `ap-southeast-1` |
| Object Ownership | Bucket owner enforced |
| ACL | Disabled |
| Block Public Access | Enabled toàn bộ |
| Bucket Versioning | Enabled |
| Default encryption | SSE-S3 |
| Object Lock | Disabled |

Thực hiện việc tạo bucket bằng tài khoản root hoặc tài khoản quản trị.

## 5.13.4. Tạo S3 bucket

### Bước 1: Mở Amazon S3

Trong AWS Management Console, tìm kiếm:

```text
Amazon S3
```

Mở:

```text
Amazon S3
→ General purpose buckets
→ Create bucket
```

### Bước 2: Chọn Region và tên bucket

Trong phần **General configuration**, nhập:

| Thuộc tính | Giá trị |
|---|---|
| AWS Region | `Asia Pacific (Singapore) ap-southeast-1` |
| Bucket type | `General purpose` |
| Bucket name | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |

Không chọn tùy chọn sao chép cấu hình từ bucket khác.

Nếu AWS báo:

```text
Bucket with the same name already exists
```

hãy kiểm tra lại account ID hoặc bổ sung một chuỗi duy nhất vào cuối tên bucket.

### Bước 3: Cấu hình Object Ownership

Trong phần **Object Ownership**, chọn:

```text
ACLs disabled
Bucket owner enforced
```

Với cấu hình này:

- ACL không được sử dụng.
- Quyền truy cập được kiểm soát bằng IAM policy và bucket policy.
- AWS account sở hữu bucket cũng sở hữu các object được tải lên.

Đây là cấu hình mặc định và phù hợp với workshop.

### Bước 4: Chặn truy cập công khai

Trong phần **Block Public Access settings for this bucket**, giữ nguyên:

```text
Block all public access
```

Đảm bảo cả bốn tùy chọn đều được bật.

Các báo cáo nhà thông minh có thể chứa thời gian hoạt động, trạng thái cửa và dữ liệu cảm biến. Vì vậy, bucket không được cấu hình public.

Không bật S3 static website hosting cho bucket này.

### Bước 5: Bật Bucket Versioning

Trong phần **Bucket Versioning**, chọn:

```text
Enable
```

Lambda sẽ ghi báo cáo theo đường dẫn cố định của từng ngày. Nếu báo cáo được chạy lại, Versioning giúp giữ phiên bản trước thay vì mất hoàn toàn file cũ.

Ví dụ, cùng một object key:

```text
reports/home01/esp32-01/2026/09/14/report.json
```

có thể có nhiều version khác nhau sau nhiều lần xuất báo cáo.

### Bước 6: Cấu hình mã hóa

Trong phần **Default encryption**, chọn:

```text
Server-side encryption with Amazon S3 managed keys
SSE-S3
```

Không cần tạo customer managed KMS key cho workshop này.

SSE-S3 cung cấp mã hóa dữ liệu lưu trữ mà không yêu cầu quản lý thêm KMS key hoặc cấp quyền KMS cho Lambda.

### Bước 7: Các tùy chọn còn lại

Giữ các thiết lập sau:

| Thuộc tính | Giá trị |
|---|---|
| Tags | Tùy chọn |
| Object Lock | Disabled |
| Advanced settings | Mặc định |

Có thể thêm tag để nhận diện project:

| Key | Value |
|---|---|
| `Project` | `SmartHome` |
| `Environment` | `Workshop` |

Chọn:

```text
Create bucket
```

![Tạo S3 bucket lưu báo cáo](/images/5.13.4.png)



## 5.13.5. Kiểm tra bucket

Mở bucket vừa tạo:

```text
Amazon S3
→ General purpose buckets
→ smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

Kiểm tra tab **Properties**:

| Thuộc tính | Kết quả cần có |
|---|---|
| AWS Region | Asia Pacific (Singapore) |
| Bucket Versioning | Enabled |
| Default encryption | SSE-S3 |
| Object Lock | Disabled |

Tiếp tục mở tab **Permissions** và kiểm tra:

```text
Block all public access: On
```

![Thuộc tính của S3 bucket](/images/5.13.5.png)



## 5.13.6. Cấp quyền xem bucket cho IAM user

Bucket được tạo bằng root hoặc tài khoản quản trị. Để IAM user `dinh-fcj` có thể kiểm tra và tải báo cáo, chúng ta tạo một policy giới hạn trong bucket này.

Mở:

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
      "Sid": "ListS3BucketsInConsole",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ViewSmartHomeReportsBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1"
    },
    {
      "Sid": "ReadSmartHomeReports",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1/*"
    }
  ]
}
```

Thay `<ACCOUNT_ID>` bằng AWS account ID thực tế.

Đặt tên policy:

```text
SmartHomeReportsDeveloperPolicy
```

Chọn:

```text
Create policy
```

Sau đó gắn policy vào nhóm:

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
SmartHomeReportsDeveloperPolicy
```

Policy này cho phép IAM user:

- Xem bucket trong AWS Console.
- Xem cấu hình Region và Versioning.
- Liệt kê các báo cáo.
- Mở hoặc tải report.
- Xem các version của report.

Policy không cho phép IAM user:

- Công khai bucket.
- Xóa bucket.
- Tắt Versioning.
- Xóa báo cáo.
- Thay đổi bucket policy.

## 5.13.7. Phân biệt quyền IAM user và quyền Lambda

Policy vừa tạo chỉ dành cho IAM user `dinh-fcj`.

Lambda `SmartHomeExport` sẽ sử dụng một execution role riêng. Role đó cần quyền ghi object:

```text
s3:PutObject
```

Quyền này sẽ được cấu hình trong phần tạo Lambda xuất báo cáo.

| Danh tính | Mục đích |
|---|---|
| IAM user `dinh-fcj` | Xem và tải báo cáo |
| `SmartHomeExportRole` | Cho phép Lambda tạo file báo cáo |
| S3 bucket policy | Kiểm soát quyền truy cập ở cấp bucket nếu cần |

Không nên cấp quyền `s3:*` cho toàn bộ bucket nếu Lambda chỉ cần ghi và đọc một số object cụ thể.

## 5.13.8. Cấu trúc lưu báo cáo

Lambda sẽ tạo object theo cấu trúc:

```text
reports/
└── home01/
    └── esp32-01/
        └── YYYY/
            └── MM/
                └── DD/
                    ├── report.json
                    └── report.csv
```

Ví dụ báo cáo ngày 14 tháng 9 năm 2026:

```text
reports/home01/esp32-01/2026/09/14/report.json
reports/home01/esp32-01/2026/09/14/report.csv
```

Ý nghĩa:

| Thành phần | Mục đích |
|---|---|
| `reports/` | Prefix gốc của báo cáo |
| `home01/` | Phân loại theo ngôi nhà |
| `esp32-01/` | Phân loại theo thiết bị |
| `2026/09/14/` | Phân loại theo ngày |
| `report.json` | Báo cáo có cấu trúc cho ứng dụng |
| `report.csv` | Báo cáo dạng bảng để mở bằng Excel |

Trong Amazon S3, các thư mục trên thực chất là prefix của object key. Không cần tạo thủ công các thư mục này. Lambda sẽ tự tạo cấu trúc khi ghi object.

## 5.13.9. Vì sao bật Versioning?

Nếu `SmartHomeExport` được chạy nhiều lần cho cùng một ngày, Lambda sẽ ghi lại cùng object key.

Khi Versioning được bật:

- Phiên bản cũ vẫn được giữ lại.
- Phiên bản mới trở thành current version.
- Có thể tải lại phiên bản trước.
- Giảm rủi ro mất báo cáo do chạy lại Lambda.

Ví dụ:

```text
report.json
├── Version 1: chạy thử thủ công
├── Version 2: chạy lại sau khi sửa Lambda
└── Version 3: Scheduler chạy tự động
```

Versioning có thể làm tăng dung lượng lưu trữ. Vì vậy, trong môi trường thực tế nên bổ sung lifecycle rule để xóa noncurrent versions sau một khoảng thời gian phù hợp.

Workshop này chưa cần cấu hình lifecycle rule.

## 5.13.10. Kiểm tra bằng IAM user

Đăng xuất tài khoản root hoặc quản trị và đăng nhập lại bằng:

```text
dinh-fcj
```

Mở Amazon S3 và chọn bucket vừa tạo.

Kết quả mong đợi:

- IAM user nhìn thấy bucket.
- Có thể mở tab Objects.
- Có thể xem Properties cơ bản.
- Không thể thay đổi public access hoặc xóa bucket.
- Tab Objects có thể đang trống vì Lambda xuất báo cáo chưa được tạo.

Bucket trống ở bước này là kết quả bình thường.

## 5.13.11. Một số lỗi thường gặp

### Bucket name đã tồn tại

S3 bucket name phải duy nhất. Hãy sử dụng:

```text
smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

hoặc bổ sung một hậu tố duy nhất.

### Tạo bucket sai Region

Region của bucket không thể thay đổi sau khi tạo. Bucket phải nằm tại:

```text
ap-southeast-1
```

để đồng nhất với Lambda và DynamoDB.

### IAM user không nhìn thấy bucket

Kiểm tra:

- `SmartHomeReportsDeveloperPolicy` đã được tạo.
- Policy đã được gắn vào `SmartHome-Developers`.
- IAM user `dinh-fcj` thuộc đúng group.
- Policy có `s3:ListAllMyBuckets` và `s3:ListBucket`.

### IAM user mở được bucket nhưng không thấy object

Ở bước này, bucket có thể chưa chứa object. Các file báo cáo chỉ xuất hiện sau khi `SmartHomeExport` được tạo và chạy thành công.

### Bucket hiển thị Public

Không tiếp tục nếu bucket có trạng thái Public. Mở tab **Permissions** và bật lại:

```text
Block all public access
```

Không thêm bucket policy có principal `"*"`.

## 5.13.12. Kết quả đạt được

Sau khi hoàn thành phần này:

- S3 bucket lưu báo cáo đã được tạo tại Singapore.
- Block Public Access được bật.
- ACL bị vô hiệu hóa.
- Bucket Versioning được bật.
- Dữ liệu được mã hóa bằng SSE-S3.
- IAM user có quyền xem và tải báo cáo.
- Cấu trúc object key cho JSON và CSV đã được xác định.
- Bucket sẵn sàng nhận dữ liệu từ `SmartHomeExport`.

## Tài liệu tham khảo

- [Getting started with Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/GetStartedWithS3.html#creating-bucket)
- [Using S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [Blocking public access to S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Amazon S3 default encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-encryption-faq.html)