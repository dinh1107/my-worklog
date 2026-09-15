---
title: "Bảo mật tài khoản AWS và kiểm soát chi phí"
date: 2026-09-14
weight: 1
chapter: false
pre: "<b>5.1. </b>"
---

# Bảo mật tài khoản AWS và kiểm soát chi phí

Trong phần này, chúng ta sẽ bảo vệ tài khoản AWS root bằng MFA, lựa chọn Region triển khai và tạo AWS Budget để theo dõi chi phí của dự án.

Đây là bước chuẩn bị cần thiết trước khi tạo IAM user và triển khai các dịch vụ AWS.

## Mục tiêu

Sau khi hoàn thành phần này:

- MFA được bật cho tài khoản root.
- Region triển khai được chọn là Singapore.
- AWS Budget được tạo để theo dõi chi phí.
- Email cảnh báo chi phí được cấu hình.
- Tài khoản root không được sử dụng cho công việc hằng ngày.

---

## 5.1.1. Đăng nhập bằng tài khoản root

Truy cập AWS Management Console:

```text
https://console.aws.amazon.com/
```

Chọn:

```text
Sign in using root user email
```

Nhập email và mật khẩu đã sử dụng để đăng ký tài khoản AWS.

Tài khoản root có toàn quyền đối với tài nguyên, bảo mật và thông tin thanh toán. Vì vậy, tài khoản này chỉ được sử dụng để thực hiện các thiết lập quản trị ban đầu.



## 5.1.2. Bật MFA cho tài khoản root

Sau khi đăng nhập, mở:

```text
Account menu
→ Security credentials
→ Multi-factor authentication (MFA)
```

Chọn:

```text
Assign MFA device
```

Nhập tên thiết bị, ví dụ:

```text
Root-MFA
```

Chọn loại thiết bị:

```text
Authenticator app
```

Tiếp theo:

1. Mở ứng dụng Google Authenticator, Microsoft Authenticator hoặc ứng dụng TOTP tương tự.
2. Quét mã QR do AWS cung cấp.
3. Nhập hai mã MFA liên tiếp.
4. Chọn **Add MFA**.

MFA bổ sung một lớp xác thực ngoài mật khẩu, giúp giảm nguy cơ tài khoản bị truy cập trái phép.



### Kiểm tra kết quả

Thiết bị MFA phải xuất hiện trong phần **Multi-factor authentication (MFA)**.

![MFA đã được bật cho tài khoản root](/images/5.1.2.png)

---

## 5.1.3. Chọn AWS Region

Mở danh sách Region ở góc trên bên phải AWS Management Console và chọn:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

![Chọn Region Singapore](/images/5.1.3.png)

Region Singapore được sử dụng thống nhất cho các tài nguyên của dự án như:

- AWS IoT Core.
- AWS Lambda.
- Amazon DynamoDB.
- Amazon SNS.
- Amazon S3.
- Amazon EventBridge Scheduler.

Việc sử dụng cùng một Region giúp tránh tạo nhầm tài nguyên và đơn giản hóa ARN, IAM policy cũng như quá trình kết nối giữa các dịch vụ.

> Trước khi tạo tài nguyên mới, cần kiểm tra Region hiện tại là `ap-southeast-1`.

---

## 5.1.4. Tạo AWS Cost Budget

Trên thanh tìm kiếm của AWS Console, nhập:

```text
Billing and Cost Management
```

Sau đó mở:

```text
Budgets
→ Create budget
```

Tại phần **Budget setup**, chọn:

```text
Customize (advanced)
```

Tại **Budget types**, chọn:

```text
Cost budget
```

### Cấu hình ngân sách

Điền các thông tin tương ứng với Budget đã tạo trong tài khoản.

| Thuộc tính | Giá trị cấu hình |
|---|---|
| Budget name | `SmartHome-Monthly-Budget` |
| Period | `Monthly` |
| Renewal type | `Recurring budget` |
| Budgeting method | `Fixed` |
| Budget amount | Giới hạn chi phí theo kế hoạch |
| Budget scope | All AWS services |



### Giải thích

- **Monthly:** Theo dõi chi phí theo từng tháng.
- **Recurring budget:** Tự động làm mới ngân sách vào đầu chu kỳ.
- **Fixed:** Sử dụng cùng một giới hạn chi phí cho mỗi tháng.
- **All AWS services:** Theo dõi chi phí của toàn bộ tài khoản.

AWS Budget chỉ theo dõi và cảnh báo chi phí. Dịch vụ này không tự động dừng tài nguyên khi đạt giới hạn nếu không cấu hình thêm Budget Actions.

---

## 5.1.5. Cấu hình cảnh báo chi phí

Tại trang **Configure alerts**, chọn:

```text
Add an alert threshold
```

Có thể sử dụng cấu hình sau:

| Cảnh báo | Ngưỡng | Loại |
|---|---:|---|
| Cảnh báo sớm | `80%` | Actual |
| Cảnh báo vượt ngân sách | `100%` | Forecasted |

Trong phần **Email recipients**, nhập email nhận cảnh báo.



Trong đó:

- **Actual** dựa trên chi phí đã phát sinh.
- **Forecasted** dựa trên chi phí AWS dự báo đến cuối tháng.



Sau khi cấu hình, chọn **Next**, kiểm tra lại thông tin và chọn:

```text
Create budget
```

---

## 5.1.6. Kiểm tra kết quả

Budget được tạo thành công khi xuất hiện trong danh sách AWS Budgets với các thông tin:

```text
Budget type: Cost
Period: Monthly
Alerts: Configured
```

![AWS Budget đã được tạo thành công](/images/5.1.6.png)

Dữ liệu chi phí không cập nhật theo thời gian thực. Có thể cần chờ một khoảng thời gian trước khi chi phí mới xuất hiện trong Budget.

Sau khi hoàn thành:

1. Đăng xuất tài khoản root.
2. Không sử dụng root để tạo tài nguyên hằng ngày.
3. Chuyển sang IAM user `dinh-fcj` trong phần tiếp theo.

---

## Kiểm tra hoàn thành

- [ ] MFA đã được bật cho root user.
- [ ] Region đã được chọn là `ap-southeast-1`.
- [ ] Cost Budget đã được tạo.
- [ ] Email cảnh báo đã được cấu hình.
- [ ] Ảnh chụp không chứa thông tin bí mật.
- [ ] Đã đăng xuất tài khoản root.

## Kết luận

Trong phần này, chúng ta đã bảo vệ tài khoản root bằng MFA, chọn Region Singapore và tạo AWS Budget để kiểm soát chi phí.

Tiếp theo, chúng ta sẽ tạo IAM group, IAM user `dinh-fcj` và cấu hình quyền truy cập cho dự án.

## Tài liệu tham khảo

- [AWS – Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)
- [AWS – Enable MFA for the root user](https://docs.aws.amazon.com/IAM/latest/UserGuide/enable-virt-mfa-for-root.html)
- [AWS – Creating a cost budget](https://docs.aws.amazon.com/cost-management/latest/userguide/create-cost-budget.html)