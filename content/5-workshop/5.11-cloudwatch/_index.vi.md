---
title: "Giám sát hệ thống với CloudWatch"
date: 2026-09-14
weight: 11
chapter: false
pre: "<b>5.11. </b>"
---

# Giám sát Lambda bằng Amazon CloudWatch

Sau khi AWS IoT Rules gọi `SmartHomeIngest`, chúng ta cần kiểm tra Lambda có được thực thi thành công hay không, thời gian xử lý bao lâu và có phát sinh lỗi trong quá trình ghi DynamoDB hoặc gửi SNS hay không.

Amazon CloudWatch cung cấp hai thành phần chính:

| Thành phần | Mục đích |
|---|---|
| CloudWatch Metrics | Theo dõi số lần gọi, lỗi, thời gian xử lý và throttling |
| CloudWatch Logs | Xem log chi tiết của từng lần Lambda thực thi |

Trong phần này, chúng ta sẽ kiểm tra metrics, mở log của `SmartHomeIngest` và thiết lập thời gian lưu log phù hợp.

## 5.11.1. Vai trò của Amazon CloudWatch

CloudWatch giúp quan sát hoạt động của hệ thống mà không cần truy cập trực tiếp vào thiết bị IoT.

Luồng giám sát hiện tại:

```text
AWS IoT Rule
→ SmartHomeIngest
→ CloudWatch Metrics và Logs
```

CloudWatch hỗ trợ trả lời các câu hỏi:

- Lambda có được AWS IoT Rule gọi hay không?
- Lambda có xử lý thành công không?
- Có lỗi `AccessDeniedException` hay không?
- Hàm có vượt quá timeout không?
- Mỗi lần xử lý mất bao lâu?
- Payload nào đã được Lambda tiếp nhận?

CloudWatch chỉ giám sát và ghi log. Dữ liệu nghiệp vụ chính vẫn được lưu trong DynamoDB.

## 5.11.2. Thông tin cần kiểm tra

Kiểm tra Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Các tài nguyên được sử dụng:

| Thành phần | Giá trị |
|---|---|
| Lambda function | `SmartHomeIngest` |
| Execution role | `SmartHomeIngestRole` |
| Log group | `/aws/lambda/SmartHomeIngest` |
| Khuyến nghị log retention | `30 days` |

Log group thường được Lambda tự động tạo sau lần thực thi đầu tiên. Không cần tạo thủ công nếu execution role đã có quyền phù hợp.

## 5.11.3. Kiểm tra quyền ghi CloudWatch Logs

Lambda cần ba quyền sau để tạo và ghi log:

```text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
```

Các quyền này được cung cấp trong AWS managed policy:

```text
AWSLambdaBasicExecutionRole
```

Thực hiện bước kiểm tra bằng root hoặc tài khoản quản trị.

Mở:

```text
IAM
→ Roles
→ SmartHomeIngestRole
→ Permissions
```

Kiểm tra danh sách permission policies. Nếu đã có:

```text
AWSLambdaBasicExecutionRole
```

thì không cần thay đổi.

Nếu chưa có, chọn:

```text
Add permissions
→ Attach policies
```

Tìm kiếm và chọn:

```text
AWSLambdaBasicExecutionRole
```

Sau đó chọn:

```text
Add permissions
```

Policy này chỉ cung cấp các quyền ghi log cơ bản. Nó không cấp quyền truy cập DynamoDB hoặc SNS.

Sau khi kiểm tra xong, đăng xuất tài khoản root và tiếp tục bằng IAM user `dinh-fcj`.

## 5.11.4. Xem các chỉ số của Lambda

Mở:

```text
AWS Lambda
→ Functions
→ SmartHomeIngest
→ Monitor
```

Trong phần **Metrics**, chọn khoảng thời gian phù hợp, ví dụ:

```text
Last 1 hour
```

hoặc:

```text
Last 3 hours
```

Kiểm tra các chỉ số:

| Metric | Ý nghĩa | Kết quả mong đợi |
|---|---|---|
| Invocations | Tổng số lần Lambda được gọi | Tăng khi có telemetry hoặc event |
| Errors | Số lần thực thi thất bại | `0` |
| Duration | Thời gian xử lý | Nhỏ hơn timeout |
| Throttles | Số lần bị giới hạn | `0` |
| Concurrent executions | Số lần thực thi đồng thời | Ở mức thấp trong workshop |

Telemetry được gửi khoảng 30 giây một lần. Vì vậy, biểu đồ **Invocations** phải tăng đều khi hệ thống hoạt động.

Các sự kiện như `gas_alarm_started` hoặc `door_brute_force_detected` sẽ tạo thêm invocation ngoài các lần xử lý telemetry.

CloudWatch Metrics có thể cập nhật chậm vài phút. Không nên kết luận hệ thống lỗi ngay khi biểu đồ chưa thay đổi.

![Các chỉ số của SmartHomeIngest](/images/5.11.4.png)


## 5.11.5. Mở CloudWatch Logs

Trong tab **Monitor** của Lambda, chọn:

```text
View CloudWatch logs
```

Hoặc truy cập trực tiếp:

```text
Amazon CloudWatch
→ Logs
→ Log groups
→ /aws/lambda/SmartHomeIngest
```

Trong log group, danh sách **Log streams** được hiển thị. Log stream mới nhất thường nằm ở đầu danh sách khi sắp xếp theo **Last event time**.

Chọn log stream mới nhất để xem chi tiết.

Mỗi lần Lambda thực thi thành công thường có các dòng hệ thống:

```text
START RequestId: ...
...
END RequestId: ...
REPORT RequestId: ...
```

Ý nghĩa:

| Dòng log | Ý nghĩa |
|---|---|
| `START` | Lambda bắt đầu xử lý request |
| Log tùy chỉnh | Dữ liệu được ghi bằng `print()` hoặc logger trong mã nguồn |
| `END` | Lambda kết thúc request |
| `REPORT` | Thời gian chạy, bộ nhớ sử dụng và thông tin tính phí |

Ví dụ phần `REPORT`:

```text
REPORT RequestId: ...
Duration: 120.00 ms
Billed Duration: 121 ms
Memory Size: 128 MB
Max Memory Used: 75 MB
```

Nếu mã nguồn có ghi log payload hoặc kết quả xử lý, có thể kiểm tra thêm:

- `recordType` là `telemetry` hoặc `event`.
- `messageId` của bản tin.
- Kết quả ghi vào DynamoDB.
- Kết quả gửi thông báo SNS.
- Thông báo bỏ qua dữ liệu trùng lặp.

![Log stream của SmartHomeIngest](/images/5.11.5.png)



## 5.11.6. Tìm kiếm lỗi trong log stream

Có thể sử dụng ô tìm kiếm để tìm các từ khóa:

```text
ERROR
Exception
AccessDenied
Task timed out
```

Các lỗi thường gặp:

| Lỗi | Nguyên nhân có thể |
|---|---|
| `AccessDeniedException` | Execution role thiếu quyền DynamoDB hoặc SNS |
| `ResourceNotFoundException` | Sai tên bảng, topic hoặc Region |
| `ConditionalCheckFailedException` | Bản tin có `messageId` đã được lưu |
| `Task timed out` | Lambda xử lý lâu hơn timeout |
| `KeyError` | Payload thiếu trường mà mã nguồn yêu cầu |
| `JSONDecodeError` | Payload không phải JSON hợp lệ |

Trong project này, `ConditionalCheckFailedException` có thể xuất hiện khi cùng một `messageId` được gửi lại. Nếu mã nguồn đã xử lý trường hợp chống trùng, đây không phải lỗi hệ thống nghiêm trọng.

## 5.11.7. Thiết lập thời gian lưu log

CloudWatch Logs mặc định có thể lưu dữ liệu không giới hạn thời gian. Đối với workshop, nên đặt thời gian lưu là 30 ngày để kiểm soát dung lượng và chi phí.

Mở:

```text
Amazon CloudWatch
→ Logs
→ Log groups
```

Tìm:

```text
/aws/lambda/SmartHomeIngest
```

Trong cột **Retention**, chọn giá trị hiện tại, sau đó đặt:

```text
30 days
```

Chọn:

```text
Save
```

Cấu hình này không làm Lambda ngừng ghi log. CloudWatch chỉ tự động loại bỏ log cũ hơn thời gian lưu đã chọn.

Không cần chụp riêng bước này. Chỉ cần ghi lại giá trị retention trong nội dung báo cáo.

## 5.11.8. Phân biệt Metrics và Logs

| Trường hợp | Nên kiểm tra |
|---|---|
| Muốn biết Lambda có được gọi không | Invocations |
| Muốn biết có bao nhiêu lần thất bại | Errors |
| Muốn biết hàm chạy trong bao lâu | Duration |
| Muốn biết nguyên nhân lỗi | CloudWatch Logs |
| Muốn xem payload đã nhận | Log tùy chỉnh |
| Muốn kiểm tra thiếu quyền | Tìm `AccessDeniedException` |
| Muốn kiểm tra timeout | Tìm `Task timed out` |

Metrics cung cấp cái nhìn tổng quan, còn Logs cung cấp thông tin chi tiết của từng lần thực thi.

## 5.11.9. Kiểm tra nhanh

Khi hệ thống gửi một telemetry mới:

1. AWS IoT Rule gọi `SmartHomeIngest`.
2. Chỉ số **Invocations** tăng.
3. Chỉ số **Errors** giữ ở `0`.
4. Một log event mới xuất hiện trong log stream.
5. Log kết thúc bằng `END` và `REPORT`.
6. DynamoDB nhận được item mới.

Khi có event cảnh báo:

1. Lambda nhận `recordType` bằng `event`.
2. Event được lưu vào DynamoDB.
3. Lambda gọi SNS.
4. CloudWatch ghi lại quá trình xử lý.
5. Email cảnh báo được gửi đến subscription đã xác nhận.

Phần kiểm thử đầu cuối sẽ được thực hiện chi tiết trong mục 5.12.

## 5.11.10. Một số lỗi thường gặp

### Không tìm thấy log group

Kiểm tra:

- Lambda đã được gọi ít nhất một lần.
- Region đang là `ap-southeast-1`.
- `SmartHomeIngestRole` có `AWSLambdaBasicExecutionRole`.
- Chờ khoảng 5–10 phút rồi tải lại trang.

### Có log group nhưng không có log stream mới

Kiểm tra:

- AWS IoT Rules đang ở trạng thái `Enabled`.
- Rule action trỏ đến `SmartHomeIngest`.
- MQTT topic khớp với câu lệnh SQL của rule.
- Khoảng thời gian đang xem trong CloudWatch có phù hợp không.

### IAM user không mở được CloudWatch Logs

Nếu xuất hiện `AccessDenied`, tài khoản quản trị cần kiểm tra quyền xem log của IAM user hoặc nhóm `SmartHome-Developers`.

Quyền xem log và quyền Lambda ghi log là hai loại quyền khác nhau:

- `SmartHomeIngestRole` cho phép Lambda ghi log.
- IAM policy của `dinh-fcj` cho phép người dùng mở và đọc log trên Console.

### Invocations tăng nhưng Errors cũng tăng

Mở log stream mới nhất và tìm:

```text
ERROR
Exception
AccessDenied
```

Sau đó xử lý lỗi theo thông báo cụ thể thay vì chỉ dựa vào biểu đồ.

## 5.11.11. Kết quả đạt được

Sau khi hoàn thành phần này:

- `SmartHomeIngest` có quyền ghi CloudWatch Logs.
- Các chỉ số Invocations, Errors và Duration được theo dõi.
- Log group `/aws/lambda/SmartHomeIngest` đã có dữ liệu.
- Có thể kiểm tra chi tiết từng lần Lambda thực thi.
- Thời gian lưu log được đặt thành 30 ngày.
- CloudWatch hỗ trợ xác định lỗi của DynamoDB, SNS và Lambda.

## Tài liệu tham khảo

- [Sending Lambda logs to CloudWatch Logs](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html)
- [Using CloudWatch metrics with Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics.html)
- [Working with CloudWatch log groups and streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)