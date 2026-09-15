---
title: "Tạo IAM user và cấu hình quyền truy cập"
date: 2026-09-14
weight: 2
chapter: false
pre: "<b>5.2. </b>"
---

# Tạo IAM user và cấu hình quyền truy cập

Trong phần này, chúng ta sẽ tạo IAM group `SmartHome-Developers`, IAM user `dinh-fcj`, thiết lập quyền truy cập AWS Management Console và bật MFA cho người dùng.

IAM user sẽ được sử dụng thay cho tài khoản root trong các bước triển khai tiếp theo.

## Tại sao cần sử dụng IAM?

Tài khoản root có toàn quyền đối với tài khoản AWS và không nên được sử dụng cho công việc hằng ngày.

AWS Identity and Access Management cho phép:

- Tạo danh tính riêng cho người triển khai.
- Quản lý quyền truy cập bằng policy.
- Nhóm những người dùng có cùng nhiệm vụ.
- Giới hạn quyền theo nguyên tắc đặc quyền tối thiểu.
- Thu hồi hoặc thay đổi quyền mà không ảnh hưởng đến root user.

Trong dự án này, quyền được quản lý thông qua IAM group thay vì gắn toàn bộ policy trực tiếp vào từng user.

## Mục tiêu

Sau khi hoàn thành:

- IAM group `SmartHome-Developers` được tạo.
- IAM user `dinh-fcj` được tạo.
- User được thêm vào đúng group.
- Quyền truy cập AWS Console được bật.
- Không tạo access key không cần thiết.
- MFA được bật cho IAM user.
- Có thể đăng nhập và sử dụng AWS bằng `dinh-fcj`.

---

## 5.2.1. Mở dịch vụ IAM

Đăng nhập AWS Management Console bằng tài khoản root hoặc tài khoản có quyền quản trị IAM.

Trên thanh tìm kiếm, nhập:

```text
IAM
```

Chọn:

```text
Identity and Access Management (IAM)
```

IAM là dịch vụ toàn cầu nên không phụ thuộc vào Region đang được chọn.

---

## 5.2.2. Tạo IAM group

Trong thanh điều hướng bên trái, chọn:

```text
User groups
→ Create group
```

Nhập tên group:

```text
SmartHome-Developers
```

Group này được sử dụng để quản lý tập trung quyền của các thành viên tham gia triển khai dự án.

Ở bước này có thể chưa gắn policy. Các policy chuyên biệt sẽ được bổ sung khi cấu hình từng dịch vụ AWS ở những phần tiếp theo.

Chọn:

```text
Create group
```

### Kiểm tra kết quả

Group phải xuất hiện trong danh sách **User groups**.

![IAM group SmartHome-Developers đã được tạo](/images/5.2.2.png)

### Tại sao sử dụng IAM group?

Khi policy được gắn vào group, tất cả thành viên trong group sẽ nhận được quyền tương ứng. Cách này giúp:

- Quản lý quyền tập trung.
- Tránh gắn lặp lại policy cho nhiều user.
- Dễ dàng bổ sung hoặc thu hồi quyền.
- Đảm bảo những người có cùng nhiệm vụ nhận được quyền thống nhất.

---

## 5.2.3. Tạo IAM user

Trong thanh điều hướng IAM, chọn:

```text
Users
→ Create user
```

Nhập tên user:

```text
dinh-fcj
```

Tích chọn:

```text
Provide user access to the AWS Management Console
```

Nếu giao diện yêu cầu chọn loại user, chọn:

```text
I want to create an IAM user
```

Thiết lập mật khẩu ban đầu. Có thể sử dụng mật khẩu được tạo tự động hoặc mật khẩu tùy chỉnh đáp ứng chính sách bảo mật của tài khoản.

Nên bật tùy chọn:

```text
Users must create a new password at next sign-in
```

Tùy chọn này yêu cầu người dùng đổi mật khẩu tạm thời trong lần đăng nhập đầu tiên.


Chọn **Next** để chuyển sang phần phân quyền.

---

## 5.2.4. Thêm user vào group

Tại phần **Set permissions**, chọn:

```text
Add user to group
```

Chọn group:

```text
SmartHome-Developers
```

Tiếp tục chọn **Next**, kiểm tra lại thông tin và chọn:

```text
Create user
```

### Kiểm tra kết quả

Mở:

```text
IAM
→ Users
→ dinh-fcj
```

Trong tab **Groups**, user phải là thành viên của:

```text
SmartHome-Developers
```

![IAM user dinh-fcj thuộc group SmartHome-Developers](/images/5.2.4.png)

---

## 5.2.5. Cấu hình quyền cho group

Mở:

```text
IAM
→ User groups
→ SmartHome-Developers
→ Permissions
```

Các quyền của user `dinh-fcj` sẽ được quản lý chủ yếu thông qua các customer managed policy được gắn vào group.

Trong toàn bộ Workshop, các policy sẽ được bổ sung theo từng giai đoạn:

| Nhóm quyền | Mục đích |
|---|---|
| AWS IoT Core | Quản lý Thing, certificate, policy và IoT Rules |
| AWS Lambda | Tạo và cấu hình Lambda function |
| DynamoDB | Tạo bảng và kiểm tra dữ liệu |
| Amazon SNS | Tạo topic và email subscription |
| CloudWatch | Kiểm tra log và chỉ số Lambda |
| Amazon S3 | Tạo bucket và kiểm tra báo cáo |
| EventBridge Scheduler | Tạo lịch gọi Lambda |
| IAM PassRole | Cho phép truyền đúng execution role đến dịch vụ AWS |

![Các policy được gắn vào SmartHome-Developers](/images/5.2.5.png)

### Nguyên tắc phân quyền

Các policy phải tuân theo nguyên tắc đặc quyền tối thiểu:

- Chỉ cấp những hành động cần cho dự án.
- Giới hạn quyền theo đúng tài nguyên khi có thể.
- Không sử dụng `AdministratorAccess` cho công việc hằng ngày.
- Không cấp `iam:PassRole` đối với mọi role.
- Không cấp quyền tạo hoặc quản lý access key của root user.


---

## 5.2.6. Kiểm tra thông tin xác thực của user

Mở:

```text
IAM
→ Users
→ dinh-fcj
→ Security credentials
```

Kiểm tra:

```text
Console access: Enabled
```

Workshop này thao tác chủ yếu qua AWS Management Console nên không cần tạo access key cho `dinh-fcj`.

Kết quả khuyến nghị:

```text
Access keys: None
```

Access key chỉ nên được tạo khi thực sự cần truy cập AWS bằng CLI, SDK hoặc API.

![Thông tin xác thực của IAM user](/images/5.2.6.png)



---

## 5.2.7. Bật MFA cho IAM user

Trong tab **Security credentials**, tìm phần:

```text
Multi-factor authentication (MFA)
```

Chọn:

```text
Assign MFA device
```

Nhập tên thiết bị, ví dụ:

```text
dinh-fcj-MFA
```

Chọn:

```text
Authenticator app
```

Sau đó:

1. Mở ứng dụng Authenticator.
2. Quét mã QR do AWS cung cấp.
3. Nhập hai mã MFA liên tiếp.
4. Chọn **Add MFA**.



---

## 5.2.8. Đăng nhập bằng IAM user

Đăng xuất tài khoản root và mở IAM sign-in URL của tài khoản.

Địa chỉ thường có dạng:

```text
https://<ACCOUNT-ALIAS-OR-ID>.signin.aws.amazon.com/console
```

Nhập:

```text
IAM user name: dinh-fcj
Password: mật khẩu của IAM user
MFA code: mã từ ứng dụng Authenticator
```

Nếu được yêu cầu, đổi mật khẩu tạm thời trong lần đăng nhập đầu tiên.

Sau khi đăng nhập:

1. Kiểm tra góc trên bên phải hiển thị `dinh-fcj`.
2. Chọn Region `Asia Pacific (Singapore)`.
3. Không tiếp tục sử dụng root cho các bước triển khai.



---

## Kiểm tra hoàn thành

- [ ] Group `SmartHome-Developers` đã được tạo.
- [ ] IAM user `dinh-fcj` đã được tạo.
- [ ] User thuộc group `SmartHome-Developers`.
- [ ] AWS Console access đã được bật.
- [ ] User không có access key không cần thiết.
- [ ] Các policy của dự án được quản lý qua group.
- [ ] MFA đã được bật cho IAM user.
- [ ] Có thể đăng nhập bằng `dinh-fcj`.
- [ ] Đã chọn Region `ap-southeast-1`.
- [ ] Không còn sử dụng root cho công việc hằng ngày.

## Kết luận

Trong phần này, chúng ta đã tạo IAM group `SmartHome-Developers`, IAM user `dinh-fcj` và thiết lập phương thức đăng nhập an toàn bằng MFA.

IAM user này sẽ được sử dụng để triển khai và quản lý các tài nguyên AWS trong những phần tiếp theo. Các quyền chuyên biệt sẽ được bổ sung khi cấu hình từng dịch vụ để bảo đảm nguyên tắc đặc quyền tối thiểu.

Tiếp theo, chúng ta sẽ thống nhất Region, tên tài nguyên và MQTT topic được sử dụng trong toàn bộ hệ thống.

## Tài liệu tham khảo

- [AWS – Create an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
- [AWS – Create IAM groups](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_create.html)
- [AWS – Enable MFA for an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html)
- [AWS – IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)