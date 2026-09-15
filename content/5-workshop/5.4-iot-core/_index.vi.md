---
title: "Thiết lập AWS IoT Core"
weight: 4
chapter: false
pre: "<b>5.4. </b>"
---

# Thiết lập AWS IoT Core

Trong phần này, chúng ta thiết lập AWS IoT Core để tiếp nhận dữ liệu từ IoT Gateway. Home Assistant và Mosquitto sử dụng chứng chỉ X.509 để kết nối bảo mật đến AWS, sau đó gửi hai loại bản tin `telemetry` và `events`.

## 5.4.1. Mục tiêu

Sau khi hoàn thành phần này, hệ thống có các tài nguyên sau:

| Thành phần | Giá trị |
|---|---|
| AWS Region | `ap-southeast-1` – Singapore |
| IoT Thing | `ha-gateway-home01` |
| MQTT client ID | `ha-gateway-home01` |
| IoT policy | `SmartHomeGatewayPolicy` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| Giao thức | MQTT/TLS, cổng `8883` |

AWS IoT Thing đại diện cho gateway trên AWS. Chứng chỉ X.509 xác thực gateway, còn IoT policy xác định gateway được phép kết nối và gửi dữ liệu đến những topic nào.

## 5.4.2. Mở AWS IoT Core

Đăng nhập AWS Management Console bằng tài khoản IAM đã được cấp quyền và kiểm tra Region:

```text
Asia Pacific (Singapore) – ap-southeast-1
```

Tìm kiếm và mở:

```text
AWS IoT Core
```

Không sử dụng tài khoản root cho các thao tác triển khai thường ngày.

## 5.4.3. Tạo IoT Thing

Trong AWS IoT Core, mở:

```text
Manage → All devices → Things → Create things
```

Thực hiện như sau:

1. Chọn **Create single thing**.
2. Nhập Thing name: `ha-gateway-home01`.
3. Không cần cấu hình Device Shadow cho workshop này.
4. Chọn **Next**.

Tên Thing được đặt theo vai trò của Home Assistant Gateway và mã ngôi nhà `home01`, giúp dễ quản lý khi mở rộng thêm gateway.

![IoT Thing ha-gateway-home01](/images/5.4.3.png)

## 5.4.4. Tạo IoT policy

Mở:

```text
Security → Policies → Create policy
```

Nhập tên policy:

```text
SmartHomeGatewayPolicy
```

Chuyển sang chế độ JSON và nhập policy dưới đây. Thay `<ACCOUNT_ID>` bằng AWS Account ID của tài khoản triển khai.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:client/ha-gateway-home01"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": [
        "arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:topic/smarthome/home01/esp32-01/telemetry",
        "arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:topic/smarthome/home01/esp32-01/events"
      ]
    }
  ]
}
```

Policy này áp dụng nguyên tắc đặc quyền tối thiểu:

- `iot:Connect`: chỉ cho phép client ID `ha-gateway-home01` kết nối.
- `iot:Publish`: chỉ cho phép gửi dữ liệu lên hai topic của thiết bị `esp32-01`.
- Gateway không được cấp quyền quản trị AWS IoT Core hoặc gửi dữ liệu đến topic khác.



![Policy SmartHomeGatewayPolicy](/images/5.4.4.png)

## 5.4.5. Tạo chứng chỉ X.509

Trong bước cấu hình chứng chỉ cho Thing, chọn:

```text
Auto-generate a new certificate
```

Sau khi AWS tạo chứng chỉ, tải xuống các file:

```text
device.pem.crt
private.pem.key
AmazonRootCA1.pem
```

Vai trò của từng file:

| File | Chức năng |
|---|---|
| `device.pem.crt` | Chứng chỉ nhận dạng IoT Gateway |
| `private.pem.key` | Khóa riêng dùng để xác thực gateway |
| `AmazonRootCA1.pem` | Chứng chỉ CA dùng để xác minh máy chủ AWS IoT |

Kích hoạt chứng chỉ bằng tùy chọn **Active**, sau đó chọn policy hiện có:

```text
SmartHomeGatewayPolicy
```

Hoàn tất quá trình tạo Thing. Nếu chứng chỉ được tạo riêng, mở chứng chỉ và sử dụng **Attach to things** để liên kết nó với `ha-gateway-home01`.


![Chứng chỉ Active và policy đã được gắn](/images/5.4.5.png)

Quan hệ giữa ba thành phần là:

```text
IoT Thing: ha-gateway-home01
        ↕
X.509 certificate: Active
        ↕
IoT policy: SmartHomeGatewayPolicy
```

## 5.4.6. Lấy Device Data Endpoint

Trong AWS IoT Core, mở:

```text
Settings → Device data endpoint
```

Endpoint của project là:

```text
akq6wc7dn4aef-ats.iot.ap-southeast-1.amazonaws.com
```

Thông tin kết nối được sử dụng bởi Mosquitto Bridge:

| Thuộc tính | Giá trị |
|---|---|
| Host | `akq6wc7dn4aef-ats.iot.ap-southeast-1.amazonaws.com` |
| Port | `8883` |
| Protocol | MQTT over TLS 1.2 |
| Client ID | `ha-gateway-home01` |

![AWS IoT Device Data Endpoint](/images/5.4.6.png)

## 5.4.7. Kiểm tra kết quả

Việc thiết lập AWS IoT Core hoàn tất khi:

- Thing `ha-gateway-home01` đã tồn tại.
- Chứng chỉ X.509 có trạng thái `Active`.
- Chứng chỉ được liên kết với Thing `ha-gateway-home01`.
- Policy `SmartHomeGatewayPolicy` được gắn vào chứng chỉ.
- Policy cho phép publish đúng hai topic `telemetry` và `events`.
- Device Data Endpoint thuộc Region `ap-southeast-1` đã được ghi lại.

Ở phần tiếp theo, endpoint và ba file chứng chỉ sẽ được sử dụng để thiết lập kết nối MQTT/TLS từ IoT Gateway đến AWS IoT Core.

