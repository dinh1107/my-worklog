---
title: "Workshop"
date: 2026-09-14
weight: 5
chapter: false
pre: "<b>5. </b>"
---

# Workshop

Workshop này cung cấp hướng dẫn từng bước để triển khai hệ thống nhà thông minh trên AWS chỉ bằng AWS Management Console với tài khoản root và IAM user `dinh-fcj`. Nội dung bao gồm bảo mật tài khoản, IAM, IoT Core, Lambda, DynamoDB, SNS, CloudWatch, S3 và EventBridge Scheduler.

Workshop chỉ tập trung vào các thao tác trên AWS cloud. Home Assistant và ESP32 chỉ được nhắc đến như nguồn dữ liệu MQTT đã chuẩn bị sẵn. Việc thiết lập phần cứng, lập trình ESP32 và cấu hình Home Assistant không được đề cập ở đây.

## Phạm vi

Workshop bao gồm 16 mục sau đây về bảo mật tài khoản, IAM, AWS IoT Core, xử lý dữ liệu, giám sát, báo cáo và dọn dẹp tài nguyên.

- [5.1 Thiết lập bảo mật và kiểm soát chi phí](5.1-account-security/) — Securing the AWS Account and Setting Up Cost Controls
- [5.2 Tạo IAM user và phân quyền](5.2-iam/) — Creating an IAM User and Configuring Permissions
- [5.3 Chuẩn bị thông tin triển khai](5.3-deployment-information/) — Preparing the AWS Deployment Information
- [5.4 Thiết lập AWS IoT Core](5.4-iot-core/) — Setting Up AWS IoT Core
- [5.5 Kiểm tra dữ liệu MQTT](5.5-mqtt-test-client/) — Verifying MQTT Data with the MQTT Test Client
- [5.6 Tạo bảng DynamoDB](5.6-dynamodb/) — Creating the Amazon DynamoDB Table
- [5.7 Vai trò IAM cho Lambda xử lý](5.7-ingest-lambda-role/) — Creating the IAM Execution Role for the Ingest Lambda
- [5.8 Lambda SmartHomeIngest](5.8-smarthome-ingest/) — Building the SmartHomeIngest Lambda Function
- [5.9 IoT Rules](5.9-iot-rules/) — Creating AWS IoT Rules
- [5.10 Cấu hình SNS](5.10-sns/) — Configuring Amazon SNS
- [5.11 Giám sát CloudWatch](5.11-cloudwatch/) — Monitoring the System with Amazon CloudWatch
- [5.12 Kiểm thử tiếp nhận dữ liệu](5.12-ingestion-testing/) — Testing the Data Ingestion Workflow
- [5.13 Tạo S3 lưu báo cáo](5.13-s3/) — Creating an Amazon S3 Bucket for Reports
- [5.14 Lambda SmartHomeExport](5.14-smarthome-export/) — Automating Reports with EventBridge Scheduler
- [5.15 EventBridge Scheduler](5.15-eventbridge-scheduler/) — Workshop Summary and Resource Cleanup
- [5.16 Dọn dẹp tài nguyên](5.16-cleanup/) — Cleaning Up AWS Resources

## Điều kiện trước khi bắt đầu

Trước khi bắt đầu workshop, hãy hoàn thành cấu hình ESP32, Home Assistant và Mosquitto MQTT Bridge. Dashboard Home Assistant đã sẵn sàng chuyển dữ liệu telemetry và sự kiện đến AWS IoT Core qua MQTT.

## Các dịch vụ AWS sử dụng

| Dịch vụ | Mục đích |
|---|---|
| AWS IAM | Bảo mật tài khoản, quản lý user, role và policy |
| AWS IoT Core | Tiếp nhận dữ liệu MQTT từ gateway |
| AWS Lambda | Xử lý dữ liệu và tạo báo cáo |
| Amazon DynamoDB | Lưu trữ telemetry và sự kiện |
| Amazon SNS | Gửi cảnh báo email |
| Amazon CloudWatch | Giám sát hệ thống và ghi log |
| Amazon S3 | Lưu trữ báo cáo |
| Amazon EventBridge Scheduler | Lên lịch báo cáo tự động |
| AWS Budgets | Giám sát và cảnh báo chi phí |

## Phân biệt thao tác Root và IAM User

| Thao tác | Tài khoản Root | IAM User `dinh-fcj` |
|---|---|---|
| Đăng nhập AWS Console | Có | Có (sau MFA) |
| Tạo IAM user | Có | Không |
| Tạo IAM group | Có | Không |
| Tạo IAM policy | Có | Không |
| Tạo IAM role | Có | Có (có quyền) |
| Tạo AWS Budget | Có | Không |
| Tạo IoT Core resources | Có | Có (có policy) |
| Tạo Lambda function | Có | Có |
| Tạo DynamoDB table | Có | Có |
| Tạo SNS topic | Có | Có |
| Tạo S3 bucket | Có | Có |
| Tạo EventBridge Scheduler | Có | Có |

## Lưu ý không công khai

Không công khai các thông tin sau:

- AWS Account ID (trừ khi cần thiết)
- IAM sign-in URL
- Access keys và secret access keys
- IoT private key
- Mật khẩu
- Email cá nhân

