---
title: "Tạo bảng Amazon DynamoDB"
date: 2026-09-14
weight: 6
chapter: false
pre: "<b>5.6. </b>"
---

# Tạo bảng Amazon DynamoDB

Trong phần này, chúng ta sẽ tạo bảng `SmartHomeTelemetry` để lưu dữ liệu telemetry và các sự kiện của hệ thống nhà thông minh.

DynamoDB được lựa chọn vì đây là cơ sở dữ liệu NoSQL serverless, phù hợp với dữ liệu IoT dạng JSON và không yêu cầu quản lý máy chủ cơ sở dữ liệu.

## Vai trò của DynamoDB

Bảng `SmartHomeTelemetry` được sử dụng để lưu:

- Dữ liệu nhiệt độ và độ ẩm.
- Giá trị ADC của cảm biến gas.
- Trạng thái các thiết bị.
- Sự kiện cảnh báo gas.
- Sự kiện nhập sai mật khẩu.
- Thời gian tạo và thời gian hết hạn của bản ghi.

Dữ liệu sau này sẽ được Lambda `SmartHomeIngest` ghi tự động vào bảng.

## Mục tiêu

Sau khi hoàn thành:

- Bảng `SmartHomeTelemetry` được tạo.
- Partition key và sort key được cấu hình.
- Bảng sử dụng chế độ On-demand.
- Time To Live được bật với thuộc tính `expiresAt`.
- Có thể kiểm tra dữ liệu bằng Explore table items.

---

## 5.6.1. Mở Amazon DynamoDB

Đăng nhập AWS Management Console bằng IAM user:

```text
dinh-fcj
```

Kiểm tra Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

Trên thanh tìm kiếm, nhập:

```text
DynamoDB
```

Chọn:

```text
Amazon DynamoDB
```

Sau đó mở:

```text
Tables
→ Create table
```

---

## 5.6.2. Cấu hình bảng

Nhập các thông tin sau:

| Thuộc tính | Giá trị |
|---|---|
| Table name | `SmartHomeTelemetry` |
| Partition key | `deviceKey` |
| Partition key type | `String` |
| Sort key | `recordKey` |
| Sort key type | `String` |

![Cấu hình tạo bảng SmartHomeTelemetry](/images/5.6.2.png)

### Giải thích khóa chính

Bảng sử dụng khóa chính tổng hợp gồm hai thành phần:

#### Partition key: `deviceKey`

`deviceKey` dùng để nhóm các bản ghi thuộc cùng một thiết bị.

Ví dụ:

```text
home01#esp32-01
```

Các bản ghi có cùng `deviceKey` sẽ được lưu và truy vấn theo cùng một định danh thiết bị.

#### Sort key: `recordKey`

`recordKey` dùng để phân biệt và sắp xếp các bản ghi của thiết bị theo loại dữ liệu, thời gian hoặc mã bản tin.

Giá trị chính xác của `recordKey` sẽ được Lambda `SmartHomeIngest` xây dựng trong phần 5.8.

Việc kết hợp hai khóa giúp:

- Lưu nhiều bản ghi cho cùng một thiết bị.
- Truy vấn dữ liệu theo thiết bị.
- Sắp xếp dữ liệu theo thời gian.
- Phân biệt telemetry và events.
- Hạn chế ghi đè bản ghi.

---

## 5.6.3. Chọn chế độ dung lượng

Trong phần **Table settings**, chọn **Customize settings** nếu cần hiển thị đầy đủ cấu hình.

Thiết lập:

| Thuộc tính | Giá trị |
|---|---|
| Table class | `DynamoDB Standard` |
| Capacity mode | `On-demand` |
| Encryption | `AWS owned key` |
| Deletion protection | `Enabled` |

### Tại sao sử dụng On-demand?

Chế độ On-demand phù hợp với Workshop vì:

- Không cần khai báo trước Read Capacity Units.
- Không cần khai báo trước Write Capacity Units.
- Tự động điều chỉnh theo số lượng yêu cầu.
- Phù hợp với dữ liệu IoT có lưu lượng thay đổi.
- Dễ kiểm soát đối với dự án nhỏ và môi trường thử nghiệm.

Deletion protection giúp hạn chế việc xóa nhầm bảng. Khi cần dọn dẹp tài nguyên ở phần cuối Workshop, phải tắt tính năng này trước khi xóa bảng.

Sau khi kiểm tra cấu hình, chọn:

```text
Create table
```

---

## 5.6.4. Kiểm tra trạng thái bảng

Quá trình tạo bảng có thể mất một khoảng thời gian ngắn.

Chỉ tiếp tục khi trạng thái chuyển thành:

```text
Table status: Active
```

Mở trang **Overview** và kiểm tra:

```text
Table name: SmartHomeTelemetry
Partition key: deviceKey
Sort key: recordKey
Capacity mode: On-demand
Region: ap-southeast-1
```

![Bảng SmartHomeTelemetry ở trạng thái Active](/images/5.6.2.png)

Khi bảng ở trạng thái `Active`, Lambda mới có thể thực hiện các thao tác ghi và đọc dữ liệu.

---

## 5.6.5. Bật Time To Live

Mở bảng:

```text
SmartHomeTelemetry
```

Sau đó chọn:

```text
Additional settings
→ Time to Live (TTL)
→ Turn on
```

Nhập TTL attribute:

```text
expiresAt
```

Xác nhận để bật TTL.

![TTL được bật với thuộc tính expiresAt](/images/5.6.5.png)

### TTL hoạt động như thế nào?

Thuộc tính `expiresAt` chứa thời điểm bản ghi hết hạn dưới dạng:

```text
Unix epoch time theo giây
```

Ví dụ:

```text
1799967600
```

Trong DynamoDB, `expiresAt` phải có kiểu:

```text
Number
```

Không sử dụng:

- Chuỗi ngày giờ ISO.
- Kiểu String.
- Unix epoch theo millisecond.

Lambda `SmartHomeIngest` sẽ tính và thêm `expiresAt` khi ghi dữ liệu vào bảng.

DynamoDB tự động xóa bản ghi sau khi hết hạn mà không tiêu thụ write throughput cho thao tác xóa tại Region gốc.

> TTL không xóa bản ghi ngay đúng thời điểm hết hạn. Bản ghi có thể được xóa trong vòng vài ngày sau đó.

Tên thuộc tính TTL có phân biệt chữ hoa và chữ thường, vì vậy phải sử dụng chính xác:

```text
expiresAt
```

---

## 5.6.6. Kiểm tra dữ liệu trong bảng

Sau khi hoàn thành Lambda và AWS IoT Rules ở các phần tiếp theo, mở:

```text
DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
```

Chọn:

```text
Run
```

Bảng cần hiển thị các bản ghi telemetry và events.

![Dữ liệu trong bảng SmartHomeTelemetry](/images/5.6.6.png)

### Các thuộc tính cần kiểm tra

| Thuộc tính | Mục đích |
|---|---|
| `deviceKey` | Xác định thiết bị |
| `recordKey` | Xác định duy nhất và sắp xếp bản ghi |
| `messageId` | Hỗ trợ nhận biết bản tin và chống trùng |
| `recordType` | Phân biệt telemetry và event |
| `timestamp` | Thời gian bản tin được tạo |
| `expiresAt` | Thời điểm bản ghi hết hạn |
| Dữ liệu cảm biến | Nhiệt độ, độ ẩm và gas ADC |
| Trạng thái | Trạng thái cửa, đèn hoặc quạt |
| Event type | Loại sự kiện cảnh báo |

Tên một số thuộc tính có thể thay đổi theo phiên bản mã nguồn Lambda đang sử dụng.

Khi chụp ảnh, cần bảo đảm dữ liệu không chứa:

- Mật khẩu cửa.
- Email cá nhân.
- Access key hoặc secret key.
- Certificate hoặc private key.
- Thông tin đăng nhập khác.

---

## Cấu hình tổng hợp

| Thuộc tính | Giá trị |
|---|---|
| Table name | `SmartHomeTelemetry` |
| Partition key | `deviceKey` – String |
| Sort key | `recordKey` – String |
| Capacity mode | `On-demand` |
| Table class | `DynamoDB Standard` |
| Encryption | `AWS owned key` |
| Deletion protection | `Enabled` |
| TTL attribute | `expiresAt` |
| TTL data type | `Number` |
| Region | `ap-southeast-1` |

---

## Kiểm tra hoàn thành

- [ ] Bảng `SmartHomeTelemetry` đã được tạo.
- [ ] Table status là `Active`.
- [ ] Partition key là `deviceKey`.
- [ ] Sort key là `recordKey`.
- [ ] Hai khóa có kiểu `String`.
- [ ] Capacity mode là `On-demand`.
- [ ] Deletion protection đã được bật.
- [ ] TTL đã được bật với `expiresAt`.
- [ ] `expiresAt` sử dụng Unix epoch giây kiểu Number.
- [ ] Có thể mở Explore table items.
- [ ] Dữ liệu không chứa thông tin bí mật.

## Kết luận

Trong phần này, chúng ta đã tạo bảng `SmartHomeTelemetry` với khóa chính tổng hợp, chế độ On-demand và Time To Live.

Bảng đã sẵn sàng để tiếp nhận dữ liệu telemetry và events từ Lambda `SmartHomeIngest`.

Tiếp theo, chúng ta sẽ tạo IAM execution role để Lambda có quyền ghi dữ liệu vào DynamoDB, gửi thông báo SNS và ghi log vào CloudWatch.

## Tài liệu tham khảo

- [AWS – DynamoDB throughput capacity modes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/capacity-mode.html)
- [AWS – Using Time To Live in DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)
- [AWS – Enable Time To Live](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/time-to-live-ttl-how-to.html)