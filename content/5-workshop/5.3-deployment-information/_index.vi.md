---
title: "Chuẩn bị thông tin triển khai trên AWS"
date: 2026-09-14
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---

# Chuẩn bị thông tin triển khai trên AWS

Trong phần này, chúng ta sẽ kiểm tra tài khoản đăng nhập, thống nhất Region, tên tài nguyên và các MQTT topic được sử dụng trong toàn bộ hệ thống.

Việc chuẩn hóa thông tin từ đầu giúp tránh tạo nhầm tài nguyên, sử dụng sai Region hoặc cấu hình topic không đồng nhất giữa các dịch vụ.

## Mục tiêu

Sau khi hoàn thành phần này:

- IAM user `dinh-fcj` được sử dụng để triển khai.
- Region được chọn là `ap-southeast-1`.
- Tên nhà, gateway và thiết bị được thống nhất.
- Tên các tài nguyên AWS được xác định.
- Hai MQTT topic `telemetry` và `events` được xác định.
- Quy ước lưu báo cáo trên Amazon S3 được thống nhất.

---

## 5.3.1. Kiểm tra IAM user và Region

Đăng nhập AWS Management Console bằng IAM user:

```text
dinh-fcj
```

Kiểm tra tên người dùng ở góc trên bên phải để bảo đảm phiên làm việc hiện tại không phải là tài khoản root.

Tiếp theo, chọn Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

![Kiểm tra IAM user và Region triển khai](/images/5.3.1.png)


### Kết quả cần đạt

```text
IAM user: dinh-fcj
Region: Asia Pacific (Singapore)
Region code: ap-southeast-1
```

IAM là dịch vụ toàn cầu, nhưng các tài nguyên như AWS IoT Core, Lambda, DynamoDB, SNS, S3 và EventBridge Scheduler được triển khai tại Region Singapore.

---

## 5.3.2. Thống nhất định danh hệ thống

Hệ thống sử dụng ba định danh chính:

| Thành phần | Định danh | Ý nghĩa |
|---|---|---|
| Ngôi nhà | `home01` | Ngôi nhà thứ nhất trong hệ thống |
| Gateway | `ha-gateway-home01` | Gateway chuyển dữ liệu lên AWS |
| Thiết bị | `esp32-01` | Thiết bị ESP32 thứ nhất của `home01` |

### Quy ước đặt tên

Tên tài nguyên được viết bằng chữ thường và sử dụng dấu gạch ngang khi cần phân tách thành phần.

Ví dụ:

```text
ha-gateway-home01
```

Tên này thể hiện:

```text
ha-gateway + home01
```

Việc sử dụng định danh thống nhất giúp xác định dữ liệu thuộc ngôi nhà, gateway và thiết bị nào khi hệ thống được mở rộng.

---

## 5.3.3. Thống nhất tên tài nguyên AWS

Các tài nguyên chính của dự án được đặt tên như sau:

| Dịch vụ | Loại tài nguyên | Tên |
|---|---|---|
| AWS IoT Core | IoT Thing | `ha-gateway-home01` |
| Amazon DynamoDB | Table | `SmartHomeTelemetry` |
| AWS Lambda | Ingest function | `SmartHomeIngest` |
| AWS Lambda | Export function | `SmartHomeExport` |
| AWS IAM | Ingest execution role | `SmartHomeIngestRole` |
| AWS IAM | Export execution role | `SmartHomeExportRole` |
| AWS IAM | Scheduler execution role | `SmartHomeSchedulerRole` |
| Amazon EventBridge Scheduler | Schedule | `SmartHomeDailyExport` |
| Amazon S3 | Reports bucket | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |

Trong tên S3 bucket, `<ACCOUNT_ID>` đại diện cho AWS Account ID của tài khoản triển khai.

Không nên công khai Account ID thật trên website. Trong báo cáo, có thể giữ dạng:

```text
smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

### Tại sao tên S3 bucket có thêm Account ID và Region?

Tên S3 bucket phải duy nhất trên toàn bộ Amazon S3. Việc thêm Account ID và Region giúp giảm khả năng trùng tên với bucket của tài khoản khác.

---

## 5.3.4. Thống nhất MQTT topic

Hệ thống sử dụng hai MQTT topic chính.

### Telemetry topic

```text
smarthome/home01/esp32-01/telemetry
```

Topic này dùng để gửi dữ liệu trạng thái định kỳ, bao gồm:

- Nhiệt độ.
- Độ ẩm.
- Giá trị ADC của cảm biến gas.
- Trạng thái cửa.
- Trạng thái đèn.
- Trạng thái quạt.
- Thông tin hoạt động của thiết bị.

Dữ liệu telemetry được gửi định kỳ:

```text
30 giây một lần
```

### Events topic

```text
smarthome/home01/esp32-01/events
```

Topic này dùng để gửi các sự kiện cần được xử lý ngay, ví dụ:

- Phát hiện nồng độ gas vượt ngưỡng.
- Mức gas trở lại an toàn.
- Nhập sai mật khẩu cửa nhiều lần.
- Các sự kiện an toàn khác của hệ thống.

Events không cần chờ chu kỳ telemetry mà được gửi ngay khi sự kiện xảy ra.

---

## 5.3.5. Giải thích cấu trúc MQTT topic

Cấu trúc topic được thiết kế theo dạng:

```text
smarthome/{homeId}/{deviceId}/{messageType}
```

Trong đó:

| Thành phần | Giá trị | Ý nghĩa |
|---|---|---|
| `smarthome` | Cố định | Tên hệ thống |
| `{homeId}` | `home01` | Định danh ngôi nhà |
| `{deviceId}` | `esp32-01` | Định danh thiết bị |
| `{messageType}` | `telemetry` hoặc `events` | Loại bản tin |

Cấu trúc này cho phép mở rộng thêm thiết bị mà không cần thay đổi kiến trúc tổng thể.

Ví dụ, nếu bổ sung thiết bị thứ hai:

```text
smarthome/home01/esp32-02/telemetry
smarthome/home01/esp32-02/events
```

---

## 5.3.6. Phân biệt telemetry và events

| Tiêu chí | Telemetry | Events |
|---|---|---|
| Mục đích | Gửi trạng thái định kỳ | Gửi sự kiện quan trọng |
| Thời điểm gửi | Mỗi 30 giây | Ngay khi sự kiện xảy ra |
| Topic cuối | `/telemetry` | `/events` |
| Ví dụ | Nhiệt độ, độ ẩm, gas ADC | Gas cao, gas an toàn, sai mật khẩu |
| Xử lý cảnh báo | Thông thường không gửi email | Có thể kích hoạt Amazon SNS |

Việc tách hai loại bản tin giúp:

- Giảm độ trễ đối với cảnh báo.
- Dễ tạo AWS IoT Rule riêng.
- Dễ theo dõi dữ liệu trên MQTT Test Client.
- Tránh gửi email cho mọi bản tin telemetry.
- Hỗ trợ lưu trữ và phân tích dữ liệu rõ ràng hơn.

> Giá trị cảm biến gas trong dự án là giá trị ADC thô, không phải ppm vì cảm biến chưa được hiệu chuẩn bằng khí chuẩn.

---

## 5.3.7. Thống nhất cấu trúc lưu báo cáo

Báo cáo hằng ngày được lưu trong Amazon S3 theo cấu trúc:

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

Ví dụ báo cáo ngày `2026-09-14`:

```text
reports/home01/esp32-01/2026/09/14/report.json
reports/home01/esp32-01/2026/09/14/report.csv
```

Cấu trúc theo năm, tháng và ngày giúp:

- Tìm kiếm báo cáo dễ dàng.
- Tránh ghi chung tất cả file vào một vị trí.
- Hỗ trợ tải xuống và lưu trữ lâu dài.
- Dễ mở rộng cho nhiều nhà và nhiều thiết bị.

---

## Bảng tổng hợp cấu hình

| Thuộc tính | Giá trị |
|---|---|
| Region | `ap-southeast-1` |
| IAM user | `dinh-fcj` |
| Home ID | `home01` |
| Gateway ID | `ha-gateway-home01` |
| Device ID | `esp32-01` |
| Telemetry interval | `30 seconds` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| DynamoDB table | `SmartHomeTelemetry` |
| Ingest Lambda | `SmartHomeIngest` |
| Export Lambda | `SmartHomeExport` |
| Daily schedule | `SmartHomeDailyExport` |
| Scheduler time zone | `Asia/Ho_Chi_Minh` |

---

## Kiểm tra hoàn thành

- [ ] Đã đăng nhập bằng IAM user `dinh-fcj`.
- [ ] Region được chọn là `ap-southeast-1`.
- [ ] Đã thống nhất tên các tài nguyên AWS.
- [ ] Đã xác định telemetry topic.
- [ ] Đã xác định events topic.
- [ ] Đã phân biệt telemetry và events.
- [ ] Đã thống nhất cấu trúc lưu báo cáo trên S3.
- [ ] Không công khai AWS Account ID hoặc thông tin đăng nhập.

## Kết luận

Trong phần này, chúng ta đã thống nhất Region, định danh hệ thống, tên tài nguyên AWS, MQTT topic và cấu trúc lưu báo cáo.

Các thông tin này sẽ được sử dụng xuyên suốt Workshop để bảo đảm cấu hình giữa AWS IoT Core, Lambda, DynamoDB, SNS, S3 và EventBridge Scheduler được đồng nhất.

Tiếp theo, chúng ta sẽ tạo IoT Thing, certificate và IoT policy trên AWS IoT Core.