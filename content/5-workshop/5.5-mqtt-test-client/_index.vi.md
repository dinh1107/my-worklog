---
title: "Kiểm tra dữ liệu bằng MQTT Test Client"
date: 2026-09-14
weight: 5
chapter: false
pre: "<b>5.5. </b>"
---

# Kiểm tra dữ liệu bằng MQTT Test Client

Trong phần này, chúng ta sẽ sử dụng MQTT Test Client của AWS IoT Core để kiểm tra dữ liệu được gửi từ IoT Gateway.

Việc kiểm tra giúp xác nhận:

- Gateway đã kết nối thành công đến AWS IoT Core.
- Certificate và IoT policy hoạt động chính xác.
- Telemetry được gửi định kỳ.
- Các sự kiện cảnh báo được gửi ngay khi phát sinh.
- Topic và nội dung JSON đúng với thiết kế.

## Các topic cần kiểm tra

| Loại dữ liệu | MQTT topic |
|---|---|
| Telemetry | `smarthome/home01/esp32-01/telemetry` |
| Events | `smarthome/home01/esp32-01/events` |

---

## 5.5.1. Mở MQTT Test Client

Đăng nhập AWS Management Console bằng IAM user:

```text
dinh-fcj
```

Kiểm tra Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

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

MQTT Test Client cho phép subscribe các topic và hiển thị bản tin ngay khi AWS IoT Core nhận được dữ liệu.

---

## 5.5.2. Subscribe telemetry topic

Trong ô **Topic filter**, nhập chính xác:

```text
smarthome/home01/esp32-01/telemetry
```

Sau đó chọn:

```text
Subscribe
```

Topic sẽ xuất hiện trong danh sách **Subscriptions**.

> MQTT topic có phân biệt chữ hoa và chữ thường. Topic phải được nhập chính xác như cấu hình của hệ thống.

---

## 5.5.3. Subscribe events topic

Quay lại tab **Subscribe to a topic** và nhập:

```text
smarthome/home01/esp32-01/events
```

Chọn:

```text
Subscribe
```

Danh sách **Subscriptions** lúc này phải có hai topic.

![Các MQTT topic đã được subscribe](/images/5.5.3.png)

### Tại sao cần subscribe riêng hai topic?

Việc tách telemetry và events giúp:

- Theo dõi từng loại dữ liệu dễ dàng hơn.
- Phân biệt dữ liệu định kỳ với sự kiện khẩn cấp.
- Hạn chế nhầm lẫn khi tạo AWS IoT Rules.
- Kiểm tra chính xác nguồn của mỗi bản tin.

---

## 5.5.4. Kiểm tra telemetry

Chọn topic:

```text
smarthome/home01/esp32-01/telemetry
```

Chờ tối đa khoảng 30 giây để nhận bản tin tiếp theo.

Mỗi bản tin cần thể hiện các thông tin chính như:

| Trường dữ liệu | Ý nghĩa |
|---|---|
| Home ID | Xác định ngôi nhà `home01` |
| Device ID | Xác định thiết bị `esp32-01` |
| Timestamp | Thời điểm dữ liệu được tạo |
| Temperature | Nhiệt độ phòng |
| Humidity | Độ ẩm phòng |
| Gas value | Giá trị ADC thô của cảm biến gas |
| Device states | Trạng thái cửa, đèn hoặc quạt |

Cấu trúc JSON thực tế có thể chứa thêm các trường trạng thái khác tùy theo phiên bản của hệ thống.

![Bản tin telemetry trên MQTT Test Client](/images/5.5.3.png)

### Kết quả cần đạt

- Bản tin xuất hiện đúng topic.
- Payload là JSON hợp lệ.
- Home ID và Device ID chính xác.
- Nhiệt độ và độ ẩm có giá trị.
- Giá trị gas được ghi nhận dưới dạng ADC.
- Bản tin mới xuất hiện đều khoảng 30 giây một lần.

> Không gọi giá trị gas là ppm nếu cảm biến chưa được hiệu chuẩn bằng khí chuẩn.

---

## 5.5.5. Kiểm tra events

Chọn topic:

```text
smarthome/home01/esp32-01/events
```

Tạo một sự kiện mới từ hệ thống nhà thông minh đã được chuẩn bị trước. Phần tạo sự kiện tại thiết bị nằm ngoài phạm vi của Workshop AWS.

Các sự kiện đã được sử dụng trong dự án gồm:

| Event type | Ý nghĩa |
|---|---|
| `gas_alarm_started` | Giá trị gas vượt ngưỡng cảnh báo |
| `gas_alarm_cleared` | Giá trị gas trở lại mức an toàn |
| `door_brute_force_detected` | Nhập sai mật khẩu cửa nhiều lần |

Bản tin event cần xuất hiện ngay sau khi sự kiện phát sinh, không phải chờ chu kỳ telemetry 30 giây.

![Bản tin event trên MQTT Test Client](/images/5.5.5.png)

### Kết quả cần đạt

- Event xuất hiện đúng topic.
- Payload là JSON hợp lệ.
- Có thông tin nhận dạng thiết bị.
- Có loại sự kiện và thời gian xảy ra.
- Sự kiện không chứa mật khẩu cửa hoặc thông tin bí mật.
- Event được nhận gần như ngay lập tức.

> MQTT Test Client chủ yếu hiển thị những bản tin đến sau thời điểm subscribe. Nếu event không được gửi dưới dạng retained message, các event cũ sẽ không tự xuất hiện lại.

---

## 5.5.6. Sử dụng wildcard khi xử lý lỗi

Nếu không biết dữ liệu đang được gửi đến topic nào, có thể tạm thời subscribe:

```text
smarthome/home01/esp32-01/#
```

Ký tự `#` đại diện cho tất cả các cấp topic phía sau. Bộ lọc trên có thể nhận cả:

```text
smarthome/home01/esp32-01/telemetry
smarthome/home01/esp32-01/events
```

Sau khi tìm được topic chính xác, nên quay lại subscribe từng topic riêng.



---

## Xử lý khi không nhận được dữ liệu

Nếu MQTT Test Client không hiển thị bản tin, kiểm tra:

1. Region có phải `ap-southeast-1` không.
2. Topic có được nhập đúng hoàn toàn không.
3. Certificate có trạng thái `Active` không.
4. IoT policy có cho phép đúng Client ID và topic không.
5. Gateway có kết nối đến đúng Device data endpoint không.
6. Hệ thống nguồn có đang gửi dữ liệu không.

Có thể subscribe tạm thời:

```text
#
```

Nếu dữ liệu xuất hiện ở một topic khác, cần kiểm tra lại cấu hình tên topic của hệ thống.

---

## Kiểm tra hoàn thành

- [ ] Đã mở MQTT Test Client tại Region Singapore.
- [ ] Đã subscribe telemetry topic.
- [ ] Đã subscribe events topic.
- [ ] Telemetry xuất hiện đều khoảng 30 giây.
- [ ] Events xuất hiện ngay khi có sự kiện.
- [ ] Payload nhận được là JSON hợp lệ.
- [ ] Giá trị gas được mô tả là ADC thô.
- [ ] Không có mật khẩu hoặc thông tin bí mật trong payload.

## Kết luận

Trong phần này, chúng ta đã xác nhận AWS IoT Core nhận thành công cả dữ liệu telemetry định kỳ và sự kiện cảnh báo tức thời.

Kết quả này chứng minh endpoint, certificate, IoT policy và kết nối MQTT đang hoạt động đúng. Dữ liệu hiện đã sẵn sàng để được xử lý và lưu trữ bởi các dịch vụ AWS ở những phần tiếp theo.

Tiếp theo, chúng ta sẽ tạo bảng Amazon DynamoDB để lưu dữ liệu telemetry và events.

## Tài liệu tham khảo

- [AWS – View MQTT messages with the MQTT Test Client](https://docs.aws.amazon.com/iot/latest/developerguide/view-mqtt-messages.html)
- [AWS – MQTT in AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)