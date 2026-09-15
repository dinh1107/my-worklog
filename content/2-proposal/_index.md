---
title: "Proposal"
date: 2026-09-14
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# AWS-INTEGRATED SMART HOME PROJECT PROPOSAL

## 1. Executive Summary

This project develops a smart home system that uses an ESP32 to collect sensor data and control household devices. Home Assistant serves as both the management dashboard and the gateway connecting the local system to AWS.

Data is transmitted to AWS IoT Core through the MQTT protocol. AWS Lambda processes the data, Amazon DynamoDB stores historical records, Amazon SNS sends email alerts, Amazon CloudWatch records system logs, and Amazon S3 stores daily reports.

Safety-related functions, such as gas leak detection, fan, buzzer and light activation, and automatic door locking, are processed directly by the ESP32. Therefore, the essential functions remain operational even when the Internet connection is unavailable.

---

## 2. Problem Statement

### Current Problems

The current system mainly operates within the local network and has several limitations:

- Data is not centrally stored in the cloud.

- Users must access Home Assistant to monitor the system status.

- There are no remote alerts for gas leaks or repeated incorrect door password attempts.

- The system does not automatically generate data reports.

- Errors and system activities are not centrally monitored.

### Proposed Solution

Home Assistant and Mosquitto are used as a gateway to forward telemetry data and events from the ESP32 to AWS IoT Core.

AWS Lambda receives and processes the data. Amazon DynamoDB stores sensor data, Amazon SNS sends email alerts, Amazon CloudWatch records logs, and Amazon S3 stores automatically generated reports.

### Benefits

- Maintains essential safety functions during Internet outages.

- Provides long-term data storage on AWS.

- Sends remote alerts through email.

- Supports centralized monitoring through system logs.

- Automatically generates daily data reports.

- Allows additional devices and sensors to be added in the future.

---

## 3. Solution Architecture

### Solution Architecture Diagram
![Sơ đồ kiến trúc](/images/2.3.png)

The system consists of three main areas:

- **Edge devices:** ESP32, sensors, and actuators.

- **Local gateway:** Home Assistant and Mosquitto MQTT Broker.

- **Cloud platform:** AWS services for data processing, storage, alerting, monitoring, and reporting.
**Workflow**



The ESP32 collects temperature, humidity, raw gas values, and device states. Critical safety functions continue to operate locally when the Internet connection is unavailable.

Home Assistant and Mosquitto act as the IoT Gateway. They receive local MQTT data and forward telemetry and events to AWS IoT Core through MQTT/TLS.

AWS IoT Rules route incoming messages to the SmartHomeIngest Lambda function. The function validates, normalizes, and deduplicates the data before writing it to the SmartHomeTelemetry DynamoDB table.

When a hazardous event is detected, Lambda publishes an alert to Amazon SNS, which sends an email to the administrator. Execution logs and metrics are monitored through Amazon CloudWatch.

Every day at 00:05 Vietnam time, EventBridge Scheduler invokes the SmartHomeExport Lambda function. The function queries the previous day's data, generates JSON and CSV reports, and stores them in Amazon S3.

AWS IAM controls access between services by applying least-privilege permissions.

### AWS Services Used

| Service | Function |
|---|---|
| AWS IAM | Manages users, roles, and access permissions. |
| AWS IoT Core | Receives MQTT data from the gateway. |
| AWS Lambda | Processes data, handles events, and generates reports. |
| Amazon DynamoDB | Stores telemetry data and system events. |
| Amazon SNS | Sends email alerts to users. |
| Amazon CloudWatch | Records logs and monitors system activities. |
| Amazon S3 | Stores reports in JSON and CSV formats. |
| Amazon EventBridge Scheduler | Automatically generates daily reports. |
| AWS Budgets | Monitors AWS usage costs and sends budget alerts. |

### Component Design

| Component | Name |
|---|---|
| Smart home | `home01` |
| Device | `esp32-01` |
| Gateway | `ha-gateway-home01` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| DynamoDB table | `SmartHomeTelemetry` |
| Data-processing Lambda | `SmartHomeIngest` |
| SNS topic | `SmartHomeAlerts` |
| Report-generation Lambda | `SmartHomeExport` |
| AWS Region | `ap-southeast-1` |

Telemetry data is sent every 30 seconds. Gas-related events and repeated incorrect password attempts are sent immediately when they occur.

---

## 4. Technical Implementation

### Implementation Phases

| Phase | Description |
|---|---|
| 1 | Complete the sensors and control logic on the ESP32. |
| 2 | Configure Home Assistant and Mosquitto MQTT. |
| 3 | Connect the Home Assistant Gateway to AWS IoT Core. |
| 4 | Process and store data using AWS Lambda and DynamoDB. |
| 5 | Send alerts through Amazon SNS and record logs in CloudWatch. |
| 6 | Export reports to Amazon S3 using Lambda and EventBridge Scheduler. |
| 7 | Test the complete system and finalize the Workshop documentation. |

### Technical Requirements

| Category | Technologies |
|---|---|
| Programming languages | C/C++, Python, YAML, JSON, and Markdown. |
| Frameworks | Arduino Framework and Home Assistant. |
| Database | Amazon DynamoDB. |
| Cloud services | AWS IoT Core, Lambda, DynamoDB, SNS, CloudWatch, S3, and EventBridge. |
| Development tools | Arduino IDE, Visual Studio Code, AWS Management Console, Git, GitHub, and Hugo. |

---

## 5. Roadmap and Milestones

| Week | Activities | Expected Result |
|---|---|---|
| Week 1 | Research the project topic and learn about AWS. | Complete the initial system architecture. |
| Week 2 | Learn about IAM and AWS Budgets. | Complete account security and cost alerts. |
| Week 3 | Improve the ESP32 firmware and Home Assistant configuration. | Ensure the local system operates reliably. |
| Week 4 | Connect the gateway to AWS IoT Core. | AWS receives telemetry data and events. |
| Week 5 | Develop Lambda functions and configure DynamoDB. | Data is processed and stored successfully. |
| Week 6 | Configure Amazon SNS and CloudWatch. | Email alerts and system logs operate correctly. |
| Week 7 | Develop the report export function. | JSON and CSV reports are stored in Amazon S3. |
| Week 8 | Configure the schedule, test the system, and complete the documentation. | Complete the system and Workshop report. |

The main project milestones are:

- ESP32 and Home Assistant operate reliably.

- The gateway successfully connects to AWS IoT Core.

- Telemetry data and events are stored in DynamoDB.

- Alerts are sent to users through email.

- Reports are automatically generated and stored in Amazon S3.

---

## 6. Cost Estimation

### Detailed Cost Estimation

The cost estimate is based on one ESP32 device, a 30-second telemetry interval, and the Singapore Region.

| Service | Estimated Usage | Estimated Monthly Cost |
|---|---:|---:|
| AWS IoT Core | Approximately 86,500 MQTT messages | USD 0.09–0.15 |
| AWS Lambda | Approximately 86,530 invocations | USD 0–0.10 |
| Amazon DynamoDB | Approximately 86,500 records | USD 0.06–0.20 |
| Amazon SNS | Fewer than 100 emails | Less than USD 0.01 |
| Amazon CloudWatch | Less than 0.5 GB of logs | USD 0–0.25 |
| Amazon S3 | Less than 1 GB of data | Less than USD 0.03 |
| EventBridge Scheduler | Approximately 30 invocations | Nearly USD 0 |
| **Estimated total** | One experimental system | **USD 0.20–1.00 per month** |

The actual cost may be lower if the AWS account is eligible for AWS Free Tier benefits or has available AWS Credits.

Actual charges may vary depending on the AWS Region, payload size, Lambda execution time, log volume, and amount of stored data.

### Cost Control Guidelines

- Configure cost alerts using AWS Budgets.

- Use DynamoDB On-demand capacity mode.

- Maintain the telemetry interval at 30 seconds.

- Send events only when the system state changes.

- Configure an appropriate retention period for CloudWatch Logs.

- Use DynamoDB TTL to automatically remove old data.

- Delete unused experimental resources.

The recommended AWS Budget thresholds are USD 1, USD 2, and USD 5.

---

## 7. Risk Assessment

| Risk | Impact | Mitigation |
|---|---|---|
| Internet connection failure | Data cannot be sent to AWS. | Keep essential safety logic on the ESP32. |
| Gateway failure | Data cannot be forwarded to AWS. | Allow the ESP32 to continue processing locally. |
| Uncalibrated gas sensor | The sensor value may not accurately represent ppm. | Use raw ADC values and determine the threshold through testing. |
| Duplicate messages | Multiple records or alerts may be generated. | Use an `eventId` to prevent duplicate processing. |
| Exposed AWS IoT certificate | Unauthorized devices may connect to AWS. | Revoke the certificate and restrict the AWS IoT Policy. |
| Exposed door password | The physical security of the system may be compromised. | Store the password only in ESP32 NVS. |
| Excessive IAM permissions | Resources may be accessed without authorization. | Apply the principle of least privilege. |
| Unexpected AWS costs | Additional AWS charges may occur. | Use AWS Budgets and regularly check Billing. |

---

## 8. Expected Outcomes

After completion, the system is expected to achieve the following results:

- Monitor temperature, humidity, gas levels, and door status.

- Control lights, ventilation fans, buzzers, door locks, and air conditioners.

- Maintain essential safety functions during Internet outages.

- Display sensor and device statuses on the Home Assistant dashboard.

- Send telemetry data and events to AWS IoT Core.

- Store system data in Amazon DynamoDB.

- Send email alerts through Amazon SNS.

- Monitor system activities through CloudWatch Logs.

- Generate JSON and CSV reports in Amazon S3.

- Automatically generate daily reports using EventBridge Scheduler.

The final result will be a smart home system that combines edge processing, local management through Home Assistant, and AWS cloud services for storage, remote alerts, monitoring, and reporting.