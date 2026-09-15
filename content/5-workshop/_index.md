---
title: "Workshop"
date: 2026-09-14
weight: 5
chapter: false
pre: "<b>5. </b>"
---

# Workshop

This workshop provides step-by-step instructions for deploying a smart home system on AWS using only the AWS Management Console with the AWS root account and IAM user `dinh-fcj`. It covers account security, IoT Core, Lambda, DynamoDB, SNS, CloudWatch, S3, and EventBridge Scheduler.

The workshop focuses exclusively on AWS cloud operations. Home Assistant and ESP32 are mentioned only as the pre-configured MQTT data source. Hardware setup, ESP32 programming, and Home Assistant configuration are not covered here.

## Scope

This workshop includes the following 16 sections covering account security, IAM, AWS IoT Core, data processing, monitoring, reporting, and cleanup.

- [5.1 Account Security and Cost Controls](5.1-account-security/) — Securing the AWS Account and Setting Up Cost Controls
- [5.2 IAM User and Permissions](5.2-iam/) — Creating an IAM User and Configuring Permissions
- [5.3 Deployment Information](5.3-deployment-information/) — Preparing the AWS Deployment Information
- [5.4 AWS IoT Core](5.4-iot-core/) — Setting Up AWS IoT Core
- [5.5 MQTT Test Client](5.5-mqtt-test-client/) — Verifying MQTT Data with the MQTT Test Client
- [5.6 DynamoDB](5.6-dynamodb/) — Creating the Amazon DynamoDB Table
- [5.7 Ingest Lambda Role](5.7-ingest-lambda-role/) — Creating the IAM Execution Role for the Ingest Lambda
- [5.8 SmartHomeIngest Lambda](5.8-smarthome-ingest/) — Building the SmartHomeIngest Lambda Function
- [5.9 IoT Rules](5.9-iot-rules/) — Creating AWS IoT Rules
- [5.10 SNS](5.10-sns/) — Configuring Amazon SNS
- [5.11 CloudWatch](5.11-cloudwatch/) — Monitoring the System with Amazon CloudWatch
- [5.12 Data Ingestion Testing](5.12-ingestion-testing/) — Testing the Data Ingestion Workflow
- [5.13 S3](5.13-s3/) — Creating an Amazon S3 Bucket for Reports
- [5.14 SmartHomeExport](5.14-smarthome-export/) — Automating Reports with EventBridge Scheduler
- [5.15 EventBridge Scheduler](5.15-eventbridge-scheduler/) — Workshop Summary and Resource Cleanup
- [5.16 Cleanup](5.16-cleanup/) — Cleaning Up AWS Resources

## Prerequisites

Before starting this workshop, complete the ESP32 firmware, Home Assistant, and Mosquitto MQTT Bridge configuration. The Home Assistant dashboard should already forward telemetry data and events to AWS IoT Core via MQTT.

## AWS Services Used

| Service | Purpose |
|---|---|
| AWS IAM | Account security, user management, roles, and policies |
| AWS IoT Core | MQTT data ingestion from the gateway |
| AWS Lambda | Data processing and report generation |
| Amazon DynamoDB | Telemetry and event data storage |
| Amazon SNS | Email alerts and notifications |
| Amazon CloudWatch | System monitoring and logging |
| Amazon S3 | Report storage |
| Amazon EventBridge Scheduler | Automated daily report scheduling |
| AWS Budgets | Cost monitoring and alerts |

## Root vs IAM User Operations

| Operation | Root Account | IAM User `dinh-fcj` |
|---|---|---|
| Sign in to AWS Console | Yes | Yes (after MFA) |
| Create IAM user | Yes | No |
| Create IAM group | Yes | No |
| Create IAM policy | Yes | No |
| Create IAM role | Yes | Yes (with permissions) |
| Create AWS Budget | Yes | No |
| Create IoT Core resources | Yes | Yes (with policies) |
| Create Lambda function | Yes | Yes |
| Create DynamoDB table | Yes | Yes |
| Create SNS topic | Yes | Yes |
| Create S3 bucket | Yes | Yes |
| Create EventBridge Scheduler | Yes | Yes |

## Confidentiality Notes

Do not publish the following information publicly:

- AWS Account ID (unless required)
- IAM sign-in URL
- Access keys and secret access keys
- IoT private key
- Passwords
- Personal email addresses

