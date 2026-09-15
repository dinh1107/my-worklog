---
title: "Cleaning Up AWS Resources"
date: 2026-09-14
weight: 16
chapter: false
pre: "<b>5.16. </b>"
---

# Cleaning Up AWS Resources

This section cleans up all AWS resources created during the workshop to avoid ongoing charges. Resources include EventBridge schedules, Lambda functions, IoT rules, certificates, things, DynamoDB tables, SNS topics, S3 content, and IAM roles. After cleanup, the AWS account is checked for any remaining charge-generating resources.

- Disable or delete EventBridge Schedule.
- Delete Lambda functions.
- Delete IoT Rules.
- Disable and delete IoT certificate.
- Delete IoT Thing.
- Delete DynamoDB table if not needed.
- Delete SNS topic and subscription.
- Delete S3 content and bucket.
- Delete IAM roles and policies.
- Check for remaining charge-generating resources.

