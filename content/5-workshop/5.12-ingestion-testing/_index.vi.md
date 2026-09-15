---
title: "Kiểm thử toàn bộ luồng dữ liệu"
date: 2026-09-14
weight: 12
chapter: false
pre: "<b>5.12. </b>"
---

# Kiểm thử toàn bộ luồng dữ liệu

Sau khi hoàn thành AWS IoT Core, IoT Rules, Lambda, DynamoDB, SNS và CloudWatch, chúng ta cần kiểm tra hệ thống theo luồng đầu-cuối.

Mục đích của phần này là xác nhận một bản tin MQTT có thể đi qua toàn bộ kiến trúc:

```text
MQTT message
→ AWS IoT Core
→ AWS IoT Rule
→ SmartHomeIngest
→ DynamoDB
→ CloudWatch
→ SNS nếu là sự kiện cảnh báo
```

Phần kiểm thử được thực hiện trên các tài nguyên AWS đã tạo. Không cần cấu hình lại phần cứng hoặc Home Assistant.

## 5.12.1. Mục tiêu kiểm thử

Sau khi hoàn thành, cần xác nhận:

- AWS IoT Core nhận được telemetry và events.
- Hai AWS IoT Rule được kích hoạt đúng topic.
- Lambda `SmartHomeIngest` xử lý thành công payload.
- Telemetry và events được lưu vào DynamoDB.
- Event cảnh báo được gửi đến Amazon SNS.
- CloudWatch ghi lại quá trình Lambda thực thi.
- Bản tin trùng `messageId` không tạo thêm item.
- Hệ thống không có lỗi quyền hoặc timeout.

## 5.12.2. Kiểm tra tài nguyên trước khi thử nghiệm

Kiểm tra Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Xác nhận các tài nguyên:

| Thành phần | Giá trị cần có |
|---|---|
| IoT Thing | `ha-gateway-home01` |
| Telemetry rule | `SmartHomeTelemetryRule` — Enabled |
| Events rule | `SmartHomeEventsRule` — Enabled |
| Lambda | `SmartHomeIngest` |
| DynamoDB table | `SmartHomeTelemetry` — Active |
| SNS topic | `SmartHomeAlerts` |
| Email subscription | Confirmed |
| CloudWatch log group | `/aws/lambda/SmartHomeIngest` |

Nếu một tài nguyên chưa ở trạng thái hoạt động, cần xử lý trước khi tiếp tục để tránh nhầm lẫn khi xác định lỗi.

## 5.12.3. Theo dõi bản tin bằng MQTT Test Client

Mở:

```text
AWS IoT Core
→ Test
→ MQTT test client
```

Chọn tab:

```text
Subscribe to a topic
```

Nhập topic filter:

```text
smarthome/home01/esp32-01/#
```

Sau đó chọn:

```text
Subscribe
```

Ký tự `#` là wildcard nhiều cấp. Topic filter trên cho phép theo dõi cả:

```text
smarthome/home01/esp32-01/telemetry
smarthome/home01/esp32-01/events
```

Khi hệ thống đang hoạt động, telemetry sẽ xuất hiện khoảng 30 giây một lần.

Kiểm tra payload có các thông tin chính như:

- `messageId`
- Thời gian gửi
- `homeId`
- `deviceId`
- Nhiệt độ
- Độ ẩm
- Giá trị gas ADC
- Trạng thái cửa, đèn hoặc quạt

Tên trường cụ thể phải khớp với cấu trúc payload được sử dụng trong `SmartHomeIngest`.

![Bản tin trong MQTT Test Client](/images/5.12.3.png)



## 5.12.4. Kiểm thử telemetry

Nếu telemetry đang được gửi tự động, chỉ cần chọn một bản tin mới trong MQTT Test Client và ghi lại `messageId`.

Nếu cần chủ động kiểm thử, mở tab:

```text
Publish to a topic
```

Nhập topic:

```text
smarthome/home01/esp32-01/telemetry
```

Sử dụng một payload tương ứng với cấu trúc đã triển khai, ví dụ:

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-telemetry-20260914-001",
  "timestamp": "2026-09-14T20:00:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "temperature": 29.5,
  "humidity": 72,
  "gasRaw": 1020,
  "doorState": "CLOSED",
  "lightState": "OFF",
  "fanState": "OFF",
  "source": "workshop-test"
}
```

Mỗi lần kiểm thử, phải đổi `messageId` và thời gian để tránh bị Lambda xác định là bản tin trùng.

Chọn:

```text
Publish
```

### Kết quả mong đợi

1. `SmartHomeTelemetryRule` nhận bản tin.
2. Rule bổ sung:

```json
{
  "recordType": "telemetry"
}
```

3. Rule gọi Lambda `SmartHomeIngest`.
4. Lambda chuẩn hóa dữ liệu.
5. Item được ghi vào bảng `SmartHomeTelemetry`.
6. CloudWatch không ghi nhận lỗi.
7. SNS không gửi email vì đây không phải sự kiện cảnh báo.

## 5.12.5. Kiểm tra telemetry trong DynamoDB

Mở:

```text
Amazon DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
```

Có thể sử dụng ô tìm kiếm hoặc bộ lọc để tìm:

```text
messageId = test-telemetry-20260914-001
```

Nếu giao diện không cho lọc trực tiếp theo `messageId`, sắp xếp theo `recordKey` hoặc xem các item mới nhất.

Item telemetry cần có các thuộc tính chính:

| Thuộc tính | Kết quả mong đợi |
|---|---|
| `deviceKey` | Xác định `home01` và `esp32-01` |
| `recordKey` | Chứa loại bản ghi và thời gian |
| `recordType` | `telemetry` |
| `messageId` | Khớp với bản tin đã gửi |
| `temperature` | `29.5` trong payload thử |
| `humidity` | `72` trong payload thử |
| `gasRaw` | `1020` trong payload thử |
| `expiresAt` | Unix epoch giây dùng cho TTL |

Không yêu cầu `deviceKey` và `recordKey` phải có cùng giá trị với bảng trên; định dạng chính xác phụ thuộc vào mã nguồn `SmartHomeIngest`. Tuy nhiên, cả partition key và sort key đều phải tồn tại.

## 5.12.6. Kiểm thử sự kiện cảnh báo gas

Trong MQTT Test Client, chọn:

```text
Publish to a topic
```

Nhập events topic:

```text
smarthome/home01/esp32-01/events
```

Nhập payload:

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-gas-started-20260914-001",
  "timestamp": "2026-09-14T20:05:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "eventType": "gas_alarm_started",
  "severity": "critical",
  "gasRaw": 2000,
  "threshold": 1500,
  "source": "workshop-test"
}
```

Chọn:

```text
Publish
```

Đây là bản tin mô phỏng phục vụ kiểm thử cloud. Nó không đại diện cho phép đo ppm vì `gasRaw` là giá trị ADC thô.

### Kết quả mong đợi

1. `SmartHomeEventsRule` nhận bản tin.
2. Rule bổ sung `recordType` bằng `event`.
3. Lambda `SmartHomeIngest` được gọi.
4. Event được lưu vào DynamoDB.
5. Lambda publish thông báo đến `SmartHomeAlerts`.
6. Email cảnh báo được gửi đến subscription đã xác nhận.
7. CloudWatch ghi lại lần xử lý thành công.

Ảnh email cảnh báo đã được sử dụng ở mục 5.10 nên không cần chụp lại trong phần này.

## 5.12.7. Kiểm thử sự kiện gas trở lại an toàn

Publish một payload mới đến events topic:

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-gas-cleared-20260914-001",
  "timestamp": "2026-09-14T20:06:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "eventType": "gas_alarm_cleared",
  "severity": "info",
  "gasRaw": 1000,
  "threshold": 1500,
  "source": "workshop-test"
}
```

Kết quả mong đợi:

- Event được lưu trong DynamoDB.
- `recordType` bằng `event`.
- SNS gửi email thông báo mức gas đã trở lại an toàn.
- CloudWatch không xuất hiện lỗi.

Sự kiện này giúp người dùng biết tình trạng nguy hiểm đã kết thúc, thay vì chỉ nhận email khi bắt đầu cảnh báo.

## 5.12.8. Kiểm thử sự kiện nhập sai mật khẩu cửa

Publish payload sau đến:

```text
smarthome/home01/esp32-01/events
```

```json
{
  "schemaVersion": "1.0",
  "messageId": "test-door-failed-20260914-001",
  "timestamp": "2026-09-14T20:07:00+07:00",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "eventType": "door_brute_force_detected",
  "severity": "critical",
  "failedAttempts": 3,
  "source": "workshop-test"
}
```

Kết quả mong đợi:

- `SmartHomeEventsRule` gọi Lambda.
- Event được lưu trong DynamoDB.
- SNS gửi email cảnh báo an ninh.
- Email không chứa mật khẩu cửa.
- CloudWatch ghi nhận lần thực thi thành công.

Hệ thống chỉ gửi số lần nhập sai, không gửi mật khẩu hoặc các phím đã nhập.

## 5.12.9. Kiểm tra các bản ghi sự kiện

Quay lại:

```text
Amazon DynamoDB
→ Tables
→ SmartHomeTelemetry
→ Explore table items
```

Kiểm tra các item mới có:

```text
recordType = event
```

Các `eventType` cần xuất hiện:

```text
gas_alarm_started
gas_alarm_cleared
door_brute_force_detected
```

![Telemetry và events trong DynamoDB](/images/5.12.9.png)


## 5.12.10. Kiểm thử cơ chế chống trùng

Cơ chế chống trùng sử dụng `messageId` để tránh lưu cùng một bản tin nhiều lần.

Thực hiện:

1. Chọn lại payload telemetry đã sử dụng.
2. Giữ nguyên `messageId`:

```text
test-telemetry-20260914-001
```

3. Publish lại payload vào cùng telemetry topic.
4. Kiểm tra DynamoDB.

Kết quả mong đợi:

- Lambda vẫn có thể được AWS IoT Rule gọi.
- DynamoDB không tạo item thứ hai cho cùng `messageId`.
- Lambda ghi log cho biết bản tin đã tồn tại hoặc bị bỏ qua.
- Không phát sinh lỗi không kiểm soát.

Cơ chế này cần thiết vì MQTT QoS 1 có thể phân phối một bản tin nhiều hơn một lần.

Không cần chụp ảnh riêng cho bước này. Có thể sử dụng CloudWatch log ở bước tiếp theo để chứng minh.

## 5.12.11. Kiểm tra CloudWatch

Mở:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Monitor
→ View CloudWatch logs
```

Mở log stream mới nhất trong:

```text
/aws/lambda/SmartHomeIngest
```

Kiểm tra:

- Có `START RequestId`.
- Lambda nhận đúng `recordType`.
- Không có `AccessDeniedException`.
- Không có `ResourceNotFoundException`.
- Không có `Task timed out`.
- Có `END RequestId`.
- Có `REPORT RequestId`.

Đối với bản tin trùng, log cần thể hiện bản tin được bỏ qua hoặc đã tồn tại. Không nên để trường hợp chống trùng làm toàn bộ invocation thất bại.



## 5.12.12. Bảng tổng hợp kết quả

| Kịch bản | IoT Rule | DynamoDB | SNS email | CloudWatch |
|---|---|---|---|---|
| Telemetry | Thành công | Có item mới | Không gửi | Không có lỗi |
| Gas bắt đầu cảnh báo | Thành công | Có event | Đã nhận | Không có lỗi |
| Gas trở lại an toàn | Thành công | Có event | Đã nhận | Không có lỗi |
| Sai mật khẩu ba lần | Thành công | Có event | Đã nhận | Không có lỗi |
| Gửi lại cùng `messageId` | Lambda được gọi | Không tạo item trùng | Không gửi lại nếu đã bỏ qua | Có log chống trùng |

## 5.12.13. Tiêu chí hoàn thành

Phần kiểm thử được xem là đạt khi:

- [x] MQTT Test Client nhận được telemetry.
- [x] MQTT Test Client nhận được events.
- [x] `SmartHomeTelemetryRule` hoạt động.
- [x] `SmartHomeEventsRule` hoạt động.
- [x] Lambda không có invocation lỗi.
- [x] DynamoDB lưu được telemetry và events.
- [x] SNS gửi được email cảnh báo.
- [x] CloudWatch có log thực thi.
- [x] Bản tin trùng không tạo item mới.
- [x] Không có mật khẩu hoặc thông tin bí mật trong dữ liệu cloud.

## 5.12.14. Một số lỗi thường gặp

### MQTT Test Client không thấy dữ liệu

Kiểm tra:

- Region là `ap-southeast-1`.
- Topic filter là `smarthome/home01/esp32-01/#`.
- Tên topic phân biệt chữ hoa và chữ thường.
- Hệ thống nguồn đang gửi dữ liệu.

### MQTT có dữ liệu nhưng Lambda không được gọi

Kiểm tra:

- IoT Rule đang `Enabled`.
- Câu lệnh SQL sử dụng đúng topic.
- Rule action là `SmartHomeIngest`.
- Lambda resource-based policy cho phép `iot.amazonaws.com`.

### Lambda chạy nhưng DynamoDB không có item

Kiểm tra CloudWatch để tìm:

```text
AccessDeniedException
ResourceNotFoundException
KeyError
```

Đồng thời kiểm tra tên bảng và các trường bắt buộc trong payload.

### DynamoDB có event nhưng không nhận được email

Kiểm tra:

- Subscription SNS đang `Confirmed`.
- `SNS_TOPIC_ARN` đúng.
- `SmartHomeIngestRole` có quyền `sns:Publish`.
- Email không nằm trong Spam hoặc Junk.
- `eventType` được Lambda hỗ trợ.

## 5.12.15. Kết quả đạt được

Sau khi hoàn thành phần này, toàn bộ luồng dữ liệu cloud đã được xác nhận:

- AWS IoT Core tiếp nhận MQTT message.
- IoT Rules phân loại telemetry và events.
- Lambda xử lý và chuẩn hóa dữ liệu.
- DynamoDB lưu dữ liệu theo thời gian.
- SNS gửi email đối với sự kiện cảnh báo.
- CloudWatch cung cấp metrics và log.
- Cơ chế `messageId` ngăn lưu dữ liệu trùng.

Hệ thống đã đáp ứng yêu cầu tiếp nhận, xử lý, lưu trữ, cảnh báo và giám sát dữ liệu nhà thông minh trên AWS.

## Tài liệu tham khảo

- [View MQTT messages with the AWS IoT MQTT client](https://docs.aws.amazon.com/iot/latest/developerguide/view-mqtt-messages.html)
- [MQTT support in AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
- [Getting started with Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStartedDynamoDB.html)
- [Viewing CloudWatch logs for Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-view.html)