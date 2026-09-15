---
title: "Preparing the AWS Deployment Information"
date: 2026-09-14
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---

# Preparing the AWS Deployment Information

In this section, we will verify the signed-in identity and define the Region, resource names, and MQTT topics used throughout the system.

Standardizing this information helps prevent resources from being created in the wrong Region and avoids inconsistent names or topics between AWS services.

## Objectives

After completing this section:

- The `dinh-fcj` IAM user is used for deployment.
- The selected Region is `ap-southeast-1`.
- The home, gateway, and device identifiers are defined.
- AWS resource names are defined.
- The `telemetry` and `events` MQTT topics are defined.
- The Amazon S3 report structure is defined.

---

## 5.3.1. Verify the IAM user and Region

Sign in to the AWS Management Console using:

```text
dinh-fcj
```

Check the user name in the upper-right corner and ensure that the current session is not using the root user.

Select the following Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

![Verify the IAM user and deployment Region](/images/5.3.1.png)



### Expected result

```text
IAM user: dinh-fcj
Region: Asia Pacific (Singapore)
Region code: ap-southeast-1
```

IAM is a global service, but resources such as AWS IoT Core, Lambda, DynamoDB, SNS, S3, and EventBridge Scheduler are deployed in the Singapore Region.

---

## 5.3.2. Define system identifiers

The system uses three primary identifiers:

| Component | Identifier | Description |
|---|---|---|
| Home | `home01` | The first home in the system |
| Gateway | `ha-gateway-home01` | The gateway that forwards data to AWS |
| Device | `esp32-01` | The first ESP32 device in `home01` |

### Naming convention

Resource names use lowercase letters and hyphens where component separation is required.

For example:

```text
ha-gateway-home01
```

This name represents:

```text
ha-gateway + home01
```

Consistent identifiers make it possible to determine which home, gateway, and device produced each message when the system is expanded.

---

## 5.3.3. Define AWS resource names

The main project resources use the following names:

| Service | Resource type | Name |
|---|---|---|
| AWS IoT Core | IoT Thing | `ha-gateway-home01` |
| Amazon DynamoDB | Table | `SmartHomeTelemetry` |
| AWS Lambda | Ingest function | `SmartHomeIngest` |
| AWS Lambda | Export function | `SmartHomeExport` |
| AWS IAM | Ingest execution role | `SmartHomeIngestRole` |
| AWS IAM | Export execution role | `SmartHomeExportRole` |
| AWS IAM | Scheduler execution role | `SmartHomeSchedulerRole` |
| Amazon EventBridge Scheduler | Schedule | `SmartHomeDailyExport` |
| Amazon S3 | Reports bucket | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |

In the S3 bucket name, `<ACCOUNT_ID>` represents the AWS Account ID used for deployment.

The actual Account ID should not be published. Keep the following masked form in public documentation:

```text
smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

### Why does the bucket name include an Account ID and Region?

Amazon S3 bucket names must be globally unique. Adding the Account ID and Region reduces the possibility of using a name that already belongs to another AWS account.

---

## 5.3.4. Define the MQTT topics

The system uses two primary MQTT topics.

### Telemetry topic

```text
smarthome/home01/esp32-01/telemetry
```

This topic carries periodic device status data, including:

- Temperature.
- Humidity.
- Raw gas sensor ADC value.
- Door state.
- Light state.
- Fan state.
- Device operation information.

Telemetry is published:

```text
Every 30 seconds
```

### Events topic

```text
smarthome/home01/esp32-01/events
```

This topic carries events that require immediate processing, such as:

- Gas level exceeding the warning threshold.
- Gas level returning to a safe range.
- Multiple incorrect door-password attempts.
- Other safety-related system events.

Events are published immediately and do not wait for the telemetry interval.

---

## 5.3.5. Explain the MQTT topic structure

The topic structure follows:

```text
smarthome/{homeId}/{deviceId}/{messageType}
```

| Segment | Value | Description |
|---|---|---|
| `smarthome` | Fixed | System name |
| `{homeId}` | `home01` | Home identifier |
| `{deviceId}` | `esp32-01` | Device identifier |
| `{messageType}` | `telemetry` or `events` | Message category |

This structure supports additional devices without changing the overall architecture.

For example:

```text
smarthome/home01/esp32-02/telemetry
smarthome/home01/esp32-02/events
```

---

## 5.3.6. Distinguish telemetry and events

| Criterion | Telemetry | Events |
|---|---|---|
| Purpose | Periodic status reporting | Important event reporting |
| Publishing time | Every 30 seconds | Immediately |
| Topic suffix | `/telemetry` | `/events` |
| Examples | Temperature, humidity, gas ADC | High gas, gas cleared, password failures |
| Notification | Normally does not send email | Can trigger Amazon SNS |

Separating the two message categories provides:

- Lower alert latency.
- Separate AWS IoT Rules.
- Clearer MQTT Test Client monitoring.
- No email notification for every telemetry message.
- Simpler storage and data analysis.

> The gas sensor value is raw ADC data, not ppm, because the sensor has not been calibrated using a reference gas.

---

## 5.3.7. Define the report storage structure

Daily reports are stored in Amazon S3 using:

```text
reports/
└── home01/
    └── esp32-01/
        └── YYYY/
            └── MM/
                └── DD/
                    ├── report.json
                    └── report.csv
```

For example, reports for `2026-09-14` are stored at:

```text
reports/home01/esp32-01/2026/09/14/report.json
reports/home01/esp32-01/2026/09/14/report.csv
```

This date-based structure:

- Makes reports easier to locate.
- Prevents every file from being stored under one prefix.
- Supports long-term storage and downloads.
- Can be expanded for multiple homes and devices.

---

## Configuration summary

| Property | Value |
|---|---|
| Region | `ap-southeast-1` |
| IAM user | `dinh-fcj` |
| Home ID | `home01` |
| Gateway ID | `ha-gateway-home01` |
| Device ID | `esp32-01` |
| Telemetry interval | `30 seconds` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| DynamoDB table | `SmartHomeTelemetry` |
| Ingest Lambda | `SmartHomeIngest` |
| Export Lambda | `SmartHomeExport` |
| Daily schedule | `SmartHomeDailyExport` |
| Scheduler time zone | `Asia/Ho_Chi_Minh` |

---

## Completion checklist

- [ ] Signed in as the `dinh-fcj` IAM user.
- [ ] Selected the `ap-southeast-1` Region.
- [ ] Defined the AWS resource names.
- [ ] Defined the telemetry topic.
- [ ] Defined the events topic.
- [ ] Distinguished telemetry from events.
- [ ] Defined the S3 report structure.
- [ ] No AWS Account ID or sign-in information is publicly exposed.

## Conclusion

In this section, we defined the Region, system identifiers, AWS resource names, MQTT topics, and report storage structure.

These values will be used throughout the workshop to keep AWS IoT Core, Lambda, DynamoDB, SNS, S3, and EventBridge Scheduler configurations consistent.

Next, we will create the IoT Thing, certificate, and IoT policy in AWS IoT Core.