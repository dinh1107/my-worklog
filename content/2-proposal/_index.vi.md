---
title: "Bản đề xuất"
date: 2026-09-14
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# ĐỀ XUẤT DỰ ÁN NHÀ THÔNG MINH TÍCH HỢP AWS

## 1. Tóm tắt

Dự án xây dựng hệ thống nhà thông minh sử dụng ESP32 để thu thập dữ liệu cảm biến và điều khiển thiết bị. Home Assistant được sử dụng làm dashboard quản lý và gateway kết nối hệ thống trong mạng nội bộ với AWS.

Dữ liệu được gửi đến AWS IoT Core thông qua MQTT. AWS Lambda xử lý dữ liệu, DynamoDB lưu trữ lịch sử, SNS gửi email cảnh báo, CloudWatch ghi log và Amazon S3 lưu báo cáo hằng ngày.

Các chức năng an toàn như cảnh báo khí gas, điều khiển quạt, còi, đèn và tự động khóa cửa được xử lý trực tiếp trên ESP32 để hệ thống vẫn hoạt động khi mất Internet.

---

## 2. Vấn đề

### Vấn đề hiện tại

Hệ thống hiện tại chủ yếu hoạt động trong mạng nội bộ và còn một số hạn chế:

- Dữ liệu chưa được lưu trữ tập trung trên đám mây.

- Người dùng phải truy cập Home Assistant để theo dõi trạng thái.

- Chưa có cảnh báo từ xa khi phát hiện khí gas hoặc nhập sai mật khẩu nhiều lần.

- Chưa có chức năng tạo báo cáo dữ liệu tự động.

- Việc theo dõi lỗi và hoạt động của hệ thống chưa được tập trung.

### Giải pháp

Sử dụng Home Assistant và Mosquitto làm gateway để chuyển telemetry và sự kiện từ ESP32 lên AWS IoT Core.

AWS Lambda tiếp nhận và xử lý dữ liệu. DynamoDB lưu dữ liệu cảm biến, SNS gửi email cảnh báo, CloudWatch ghi log và S3 lưu các báo cáo được tạo tự động.

### Lợi ích

- Duy trì chức năng an toàn khi mất Internet.

- Lưu trữ dữ liệu lâu dài trên AWS.

- Nhận cảnh báo từ xa qua email.

- Theo dõi hoạt động hệ thống thông qua log.

- Tạo báo cáo dữ liệu tự động.

- Có thể mở rộng thêm thiết bị trong tương lai.

---

## 3. Kiến trúc giải pháp

### Sơ đồ kiến trúc giải pháp
![Sơ đồ kiến trúc](/images/2.3.png)

Hệ thống gồm ba khu vực chính:

- **Thiết bị tại biên:** ESP32, cảm biến và thiết bị chấp hành.

- **Gateway cục bộ:** Home Assistant và Mosquitto MQTT Broker.

- **Nền tảng đám mây:** Các dịch vụ AWS phục vụ xử lý, lưu trữ, cảnh báo và báo cáo.

**Luồng hoạt động**



ESP32 thu thập nhiệt độ, độ ẩm, giá trị gas và trạng thái thiết bị. Các chức năng an toàn vẫn được xử lý tại chỗ khi mất Internet.

Home Assistant và Mosquitto đóng vai trò IoT Gateway, nhận dữ liệu MQTT nội bộ rồi chuyển tiếp telemetry và events đến AWS IoT Core bằng kết nối MQTT/TLS.

AWS IoT Rules định tuyến bản tin đến Lambda SmartHomeIngest. Lambda kiểm tra, chuẩn hóa và chống lưu trùng trước khi ghi dữ liệu vào bảng SmartHomeTelemetry trên DynamoDB.

Khi phát hiện sự kiện nguy hiểm, Lambda xuất bản cảnh báo đến Amazon SNS để gửi email cho người quản trị. Nhật ký và chỉ số thực thi được theo dõi trong Amazon CloudWatch.

Mỗi ngày lúc 00:05 theo giờ Việt Nam, EventBridge Scheduler gọi Lambda SmartHomeExport. Lambda truy vấn dữ liệu của ngày hôm trước, tạo báo cáo JSON và CSV rồi lưu vào Amazon S3.

AWS IAM kiểm soát quyền truy cập giữa các dịch vụ theo nguyên tắc đặc quyền tối thiểu.

### Các dịch vụ AWS sử dụng

| Dịch vụ | Chức năng |
|---|---|
| AWS IAM | Quản lý người dùng và quyền truy cập. |
| AWS IoT Core | Tiếp nhận dữ liệu MQTT từ gateway. |
| AWS Lambda | Xử lý dữ liệu và tạo báo cáo. |
| Amazon DynamoDB | Lưu telemetry và sự kiện. |
| Amazon SNS | Gửi email cảnh báo. |
| Amazon CloudWatch | Ghi log hoạt động của hệ thống. |
| Amazon S3 | Lưu báo cáo JSON và CSV. |
| EventBridge Scheduler | Tự động xuất báo cáo hằng ngày. |
| AWS Budgets | Theo dõi và cảnh báo chi phí. |

### Thiết kế thành phần

| Thành phần | Tên |
|---|---|
| Ngôi nhà | `home01` |
| Thiết bị | `esp32-01` |
| Gateway | `ha-gateway-home01` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| DynamoDB table | `SmartHomeTelemetry` |
| Lambda xử lý | `SmartHomeIngest` |
| SNS topic | `SmartHomeAlerts` |
| Lambda báo cáo | `SmartHomeExport` |

Telemetry được gửi mỗi 30 giây. Các sự kiện khí gas và nhập sai mật khẩu được gửi ngay khi phát sinh.

---

## 4. Triển khai kỹ thuật

### Các giai đoạn triển khai

| Giai đoạn | Nội dung |
|---|---|
| 1 | Hoàn thiện cảm biến và logic điều khiển trên ESP32. |
| 2 | Cấu hình Home Assistant và Mosquitto MQTT. |
| 3 | Kết nối Home Assistant Gateway với AWS IoT Core. |
| 4 | Xử lý và lưu dữ liệu bằng Lambda và DynamoDB. |
| 5 | Gửi cảnh báo bằng SNS và ghi log bằng CloudWatch. |
| 6 | Xuất báo cáo sang S3 bằng Lambda và EventBridge Scheduler. |
| 7 | Kiểm thử và hoàn thiện tài liệu Workshop. |

### Yêu cầu kỹ thuật

| Nhóm | Công nghệ sử dụng |
|---|---|
| Ngôn ngữ lập trình | C/C++, Python, YAML, JSON và Markdown. |
| Framework | Arduino Framework và Home Assistant. |
| Cơ sở dữ liệu | Amazon DynamoDB. |
| Dịch vụ đám mây | IoT Core, Lambda, DynamoDB, SNS, CloudWatch, S3 và EventBridge. |
| Công cụ phát triển | Arduino IDE, Visual Studio Code, AWS Console, Git, GitHub và Hugo. |

---

## 5. Lộ trình và các mốc

| Tuần | Nội dung | Kết quả |
|---|---|---|
| Tuần 1 | Khảo sát đề tài và tìm hiểu AWS. | Hoàn thành kiến trúc ban đầu. |
| Tuần 2 | Tìm hiểu IAM và AWS Budgets. | Hoàn thành bảo mật tài khoản và cảnh báo chi phí. |
| Tuần 3 | Cải tiến ESP32 và Home Assistant. | Hệ thống cục bộ hoạt động ổn định. |
| Tuần 4 | Kết nối AWS IoT Core. | AWS nhận được telemetry và events. |
| Tuần 5 | Xây dựng Lambda và DynamoDB. | Dữ liệu được xử lý và lưu trữ. |
| Tuần 6 | Cấu hình SNS và CloudWatch. | Email cảnh báo và log hoạt động. |
| Tuần 7 | Xây dựng chức năng xuất báo cáo. | Báo cáo được lưu trong S3. |
| Tuần 8 | Cấu hình lịch chạy và kiểm thử. | Hoàn thành hệ thống và báo cáo. |

Các mốc chính của dự án:

- ESP32 và Home Assistant hoạt động ổn định.

- Gateway kết nối thành công với AWS IoT Core.

- Dữ liệu được lưu trong DynamoDB.

- Cảnh báo được gửi qua email.

- Báo cáo được tạo tự động trên Amazon S3.

---

## 6. Ước tính chi phí

### Ước tính chi phí chi tiết

Chi phí được ước tính với một ESP32, telemetry mỗi 30 giây và Region Singapore.

| Dịch vụ | Mức sử dụng dự kiến | Chi phí mỗi tháng |
|---|---:|---:|
| AWS IoT Core | Khoảng 86.500 bản tin | 0,09–0,15 USD |
| AWS Lambda | Khoảng 86.530 lượt gọi | 0–0,10 USD |
| Amazon DynamoDB | Khoảng 86.500 bản ghi | 0,06–0,20 USD |
| Amazon SNS | Dưới 100 email | Dưới 0,01 USD |
| Amazon CloudWatch | Dưới 0,5 GB log | 0–0,25 USD |
| Amazon S3 | Dưới 1 GB dữ liệu | Dưới 0,03 USD |
| EventBridge Scheduler | Khoảng 30 lượt gọi | Gần 0 USD |
| **Tổng dự kiến** | Một hệ thống thử nghiệm | **0,20–1,00 USD/tháng** |

Chi phí thực tế có thể thấp hơn nếu tài khoản còn AWS Free Tier hoặc AWS Credit.

### Hướng dẫn kiểm soát chi phí

- Thiết lập cảnh báo bằng AWS Budgets.

- Sử dụng DynamoDB On-demand.

- Giữ chu kỳ telemetry ở mức 30 giây.

- Chỉ gửi sự kiện khi trạng thái thay đổi.

- Thiết lập thời gian lưu CloudWatch Logs.

- Sử dụng DynamoDB TTL để xóa dữ liệu cũ.

- Xóa tài nguyên thử nghiệm không còn sử dụng.

Ngưỡng cảnh báo đề xuất là 1 USD, 2 USD và 5 USD.

---

## 7. Đánh giá rủi ro

| Rủi ro | Ảnh hưởng | Biện pháp |
|---|---|---|
| Mất Internet | Không gửi được dữ liệu lên AWS. | Giữ logic an toàn trên ESP32. |
| Gateway ngừng hoạt động | Dữ liệu không được chuyển lên AWS. | ESP32 tiếp tục xử lý tại chỗ. |
| Cảm biến gas chưa hiệu chuẩn | Giá trị không chính xác theo ppm. | Sử dụng ADC thô và ngưỡng thử nghiệm. |
| Bản tin bị gửi trùng | Có thể tạo nhiều cảnh báo. | Sử dụng `eventId` để chống trùng. |
| Lộ chứng chỉ AWS | Thiết bị lạ có thể kết nối. | Thu hồi chứng chỉ và giới hạn IoT Policy. |
| Lộ mật khẩu cửa | Ảnh hưởng an toàn hệ thống. | Chỉ lưu mật khẩu trong NVS. |
| IAM cấp quyền quá rộng | Tăng nguy cơ truy cập trái phép. | Áp dụng nguyên tắc quyền tối thiểu. |
| Chi phí tăng ngoài dự kiến | Phát sinh thanh toán AWS. | Sử dụng Budgets và kiểm tra Billing. |

---

## 8. Kết quả mong đợi

Sau khi hoàn thành, hệ thống dự kiến đạt được các kết quả sau:

- Giám sát nhiệt độ, độ ẩm, khí gas và trạng thái cửa.

- Điều khiển đèn, quạt, còi, khóa cửa và điều hòa.

- Duy trì các chức năng an toàn khi mất Internet.

- Hiển thị trạng thái thiết bị trên Home Assistant.

- Gửi telemetry và sự kiện lên AWS IoT Core.

- Lưu dữ liệu trong Amazon DynamoDB.

- Gửi email cảnh báo qua Amazon SNS.

- Theo dõi hoạt động bằng CloudWatch Logs.

- Tạo báo cáo JSON và CSV trong Amazon S3.

- Tự động xuất báo cáo bằng EventBridge Scheduler.

Kết quả cuối cùng là một hệ thống nhà thông minh kết hợp được khả năng xử lý tại biên, quản lý cục bộ bằng Home Assistant và các dịch vụ điện toán đám mây của AWS.