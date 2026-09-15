---
title: "Creating the Report Export Lambda"
date: 2026-09-14
weight: 14
chapter: false
pre: "<b>5.14. </b>"
---

# Creating the JSON and CSV Export Lambda

In the previous section, an S3 bucket was created for report storage. We will now create the `SmartHomeExport` Lambda function to:

1. Query data from DynamoDB.
2. Filter records by date.
3. Calculate telemetry statistics.
4. Count event types.
5. Generate JSON and CSV reports.
6. Store both files in Amazon S3.

The processing flow is:

```text
DynamoDB SmartHomeTelemetry
→ SmartHomeExport
→ report.json and report.csv
→ Amazon S3
```

## 5.14.1. Configuration overview

| Component | Value |
|---|---|
| Function name | `SmartHomeExport` |
| Runtime | Python 3.14 |
| Architecture | x86_64 |
| Execution role | `SmartHomeExportRole` |
| DynamoDB table | `SmartHomeTelemetry` |
| S3 bucket | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |
| Memory | 128 MB |
| Timeout | 30 seconds |
| Region | `ap-southeast-1` |

Replace `<ACCOUNT_ID>` with the actual AWS account ID.

## 5.14.2. Create the Lambda execution role

Perform this step using the root or administrator account.

Navigate to:

```text
IAM
→ Roles
→ Create role
```

Under **Trusted entity type**, select:

```text
AWS service
```

Under **Use case**, select:

```text
Lambda
```

Choose **Next** and attach:

```text
AWSLambdaBasicExecutionRole
```

This managed policy allows Lambda to write logs to CloudWatch.

Enter the role name:

```text
SmartHomeExportRole
```

Choose:

```text
Create role
```

## 5.14.3. Grant DynamoDB and S3 permissions

Open the new role:

```text
IAM
→ Roles
→ SmartHomeExportRole
→ Permissions
→ Add permissions
→ Create inline policy
→ JSON
```

Enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSmartHomeTelemetry",
      "Effect": "Allow",
      "Action": [
        "dynamodb:Query",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:ap-southeast-1:<ACCOUNT_ID>:table/SmartHomeTelemetry"
    },
    {
      "Sid": "WriteSmartHomeReports",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1/reports/*"
    },
    {
      "Sid": "ViewReportBucketLocation",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1"
    }
  ]
}
```

Name the policy:

```text
SmartHomeExportDataPolicy
```

Choose:

```text
Create policy
```

The role can:

- Read records from `SmartHomeTelemetry`.
- Write objects under the `reports/` prefix.
- Write logs to CloudWatch.

It cannot delete the table, delete the bucket, or make reports public.

## 5.14.4. Create the Lambda function

Sign out of the root account and sign in as:

```text
dinh-fcj
```

Navigate to:

```text
AWS Lambda
→ Functions
→ Create function
```

Select:

```text
Author from scratch
```

Enter:

| Property | Value |
|---|---|
| Function name | `SmartHomeExport` |
| Runtime | `Python 3.14` |
| Architecture | `x86_64` |

Under **Change default execution role**, select:

```text
Use an existing role
```

Select:

```text
SmartHomeExportRole
```

Choose:

```text
Create function
```

If an `iam:PassRole` error appears, the administrator must allow `dinh-fcj` to pass only `SmartHomeExportRole` to Lambda.

![SmartHomeExport Lambda function](/images/5.14.4.png)



## 5.14.5. Configure environment variables

First, open an existing DynamoDB item and copy its exact:

```text
deviceKey
```

Then navigate to:

```text
SmartHomeExport
→ Configuration
→ Environment variables
→ Edit
```

Add:

| Key | Value |
|---|---|
| `TABLE_NAME` | `SmartHomeTelemetry` |
| `REPORT_BUCKET` | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |
| `DEVICE_KEY` | Actual `deviceKey` value from DynamoDB |
| `HOME_ID` | `home01` |
| `DEVICE_ID` | `esp32-01` |

For example, if DynamoDB contains:

```text
HOME#home01#DEVICE#esp32-01
```

configure:

```text
DEVICE_KEY=HOME#home01#DEVICE#esp32-01
```

Do not guess the `deviceKey` format. Copy the value from an item created by `SmartHomeIngest`.

Choose:

```text
Save
```

## 5.14.6. Configure the timeout

Navigate to:

```text
SmartHomeExport
→ Configuration
→ General configuration
→ Edit
```

Set:

| Property | Value |
|---|---|
| Memory | `256 MB` |
| Timeout | `30 seconds` |

Choose:

```text
Save
```

A 30-second timeout is suitable for the workshop dataset. A larger system should optimize its DynamoDB keys or increase the timeout.

## 5.14.7. Add the source code

Open:

```text
Code
→ lambda_function.py
```

Replace the default code with the Python source shown in the Vietnamese version of this section, then choose:

```text
Deploy
```

The same source code is used for both language versions of the workshop.

## 5.14.8. How the function works

### Selecting the report date

When the event contains:

```json
{
  "reportDate": "2026-09-14"
}
```

Lambda exports the specified date.

When the event is empty:

```json
{}
```

Lambda automatically exports the previous day using:

```text
Asia/Ho_Chi_Minh
```

EventBridge Scheduler will use the empty event in the next section.

### Querying DynamoDB

The function queries records for the configured device using:

```python
Key("deviceKey").eq(DEVICE_KEY)
```

It then filters the returned records by local report date.

This approach is suitable for the workshop dataset. A larger system should design its sort key or a Global Secondary Index to query the required time range directly.

### Telemetry statistics

The function calculates:

```text
minimum
maximum
average
```

for:

- Temperature.
- Humidity.
- Raw gas ADC value.

`gasRaw` is not converted to ppm because the sensor has not been calibrated using reference gas.

### Event counts

Events are grouped by `eventType`, for example:

```json
{
  "gas_alarm_started": 1,
  "gas_alarm_cleared": 1,
  "door_brute_force_detected": 2
}
```

### S3 objects

The function writes:

```text
reports/home01/esp32-01/YYYY/MM/DD/report.json
reports/home01/esp32-01/YYYY/MM/DD/report.csv
```

JSON is suitable for applications and automated processing. CSV is suitable for Excel and spreadsheet software.

## 5.14.9. Create a test event

Navigate to:

```text
SmartHomeExport
→ Test
→ Create new event
```

Enter the event name:

```text
ExportSpecificDate
```

Use a date that contains DynamoDB data:

```json
{
  "reportDate": "2026-09-14"
}
```

Choose:

```text
Save
→ Test
```

A successful response has the following structure:

```json
{
  "statusCode": 200,
  "reportDate": "2026-09-14",
  "totalRecords": 120,
  "telemetryRecords": 117,
  "eventRecords": 3,
  "jsonKey": "reports/home01/esp32-01/2026/09/14/report.json",
  "csvKey": "reports/home01/esp32-01/2026/09/14/report.csv"
}
```

The actual record counts will depend on the stored data.

![Successful SmartHomeExport test](/images/5.14.9.png)


## 5.14.10. Verify the S3 objects

Navigate to:

```text
Amazon S3
→ smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

Open:

```text
reports/
→ home01/
→ esp32-01/
→ 2026/
→ 09/
→ 14/
```

The folder must contain:

```text
report.json
report.csv
```

Verify that:

- Both files are larger than `0 B`.
- Their **Last modified** time matches the Lambda test.
- The JSON content type is `application/json`.
- Both files can be downloaded.

![JSON and CSV reports in S3](/images/5.14.10.png)



## 5.14.11. Verify the report contents

The JSON report contains:

```json
{
  "reportDate": "2026-09-14",
  "timezone": "Asia/Ho_Chi_Minh",
  "homeId": "home01",
  "deviceId": "esp32-01",
  "summary": {
    "totalRecords": 120,
    "telemetryRecords": 117,
    "eventRecords": 3
  },
  "telemetryStatistics": {
    "temperature": {
      "minimum": 27.5,
      "maximum": 31.2,
      "average": 29.4
    },
    "humidity": {
      "minimum": 65,
      "maximum": 76,
      "average": 71.2
    },
    "gasRaw": {
      "minimum": 980,
      "maximum": 2050,
      "average": 1085.4
    }
  }
}
```

These values are examples. The report website should present values from the actual project output.

The CSV report contains columns such as:

```text
timestamp,recordType,messageId,homeId,deviceId,temperature,
humidity,gasRaw,eventType,severity,doorState,lightState,fanState
```

## 5.14.12. Verify S3 Versioning

Run the same test event again:

```json
{
  "reportDate": "2026-09-14"
}
```

Open the report folder and enable:

```text
Show versions
```

Because Versioning is enabled, S3 creates another version instead of permanently replacing the previous report.



## 5.14.13. Common issues

### AccessDenied while querying DynamoDB

Verify that:

- The execution role is `SmartHomeExportRole`.
- The role has `dynamodb:Query`.
- The resource ARN references `SmartHomeTelemetry`.
- The selected Region is `ap-southeast-1`.

### AccessDenied while writing to S3

Verify that:

- The role has `s3:PutObject`.
- The policy resource ends with `/reports/*`.
- `REPORT_BUCKET` contains the correct bucket name.

### The report contains zero records

Verify that:

- `DEVICE_KEY` exactly matches the DynamoDB item.
- `reportDate` contains existing data.
- Stored timestamps use a supported format.
- The timestamps are interpreted in the correct time zone.

A zero-record report can still be created successfully and does not indicate an S3 write failure.

### CSV text appears incorrectly in Excel

The function uses:

```python
utf-8-sig
```

to improve UTF-8 compatibility with Excel. If necessary, import the file through **Data → From Text/CSV** and select UTF-8.

### Lambda times out

If too many records are stored:

- Temporarily increase the timeout.
- Reduce the queried data range.
- Redesign the sort key for date-range queries.
- Do not replace the query with a full table `Scan`.

## 5.14.14. Result

After completing this section:

- `SmartHomeExportRole` follows least-privilege access.
- `SmartHomeExport` has been deployed.
- A specific report date can be passed through the event.
- An empty `{}` event selects the previous day in Vietnam time.
- DynamoDB records are exported to JSON and CSV.
- Both files are stored under the correct S3 prefix.
- S3 Versioning preserves previous report versions.
- CloudWatch records each export invocation.

## References

- [Creating an AWS Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
- [Querying DynamoDB tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html)
- [Amazon S3 PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html)
- [Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)