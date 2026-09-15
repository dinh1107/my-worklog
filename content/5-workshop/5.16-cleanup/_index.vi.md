---
title: "Dọn dẹp tài nguyên AWS"
date: 2026-09-14
weight: 16
chapter: false
pre: "<b>5.16. </b>"
---

# Dọn dẹp tài nguyên AWS

Mục này dọn dẹp tất cả tài nguyên AWS tạo trong workshop để tránh phát sinh chi phí. Tài nguyên bao gồm lịch EventBridge, Lambda functions, IoT rules, certificates, things, DynamoDB tables, SNS topics, S3 content và IAM roles. Sau khi dọn dẹp, tài khoản AWS được kiểm tra để đảm bảo không còn tài nguyên phát sinh chi phí.

- Tắt hoặc xóa EventBridge Schedule.
- Xóa Lambda functions.
- Xóa IoT Rules.
- Vô hiệu hóa và xóa IoT certificate.
- Xóa IoT Thing.
- Xóa DynamoDB table nếu không cần giữ.
- Xóa SNS topic và subscription.
- Xóa nội dung và bucket S3.
- Xóa IAM roles và policies của dự án.
- Kiểm tra tài nguyên còn phát sinh chi phí.
