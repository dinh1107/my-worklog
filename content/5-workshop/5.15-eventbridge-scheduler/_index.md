---
title: "Automating Daily Report Exports"
date: 2026-09-14
weight: 15
chapter: false
pre: "<b>5.15. </b>"
---

# Automating Reports with EventBridge Scheduler

In the previous section, `SmartHomeExport` generated JSON and CSV reports when manually invoked.

In this section, Amazon EventBridge Scheduler automatically invokes the function at:

```text
00:05 every day in Vietnam time
```

When Lambda receives an empty `{}` payload, it automatically selects the previous date and exports the report to Amazon S3.

The automated flow is:

```text
00:05 Asia/Ho_Chi_Minh
→ EventBridge Scheduler
→ SmartHomeExport
→ DynamoDB
→ report.json and report.csv
→ Amazon S3
```

## 5.15.1. Role of EventBridge Scheduler

EventBridge Scheduler can invoke an AWS service:

- Once at a specified time.
- At a fixed rate.
- Using a cron expression.
- In a specified time zone.

This project uses a cron-based schedule because the report must be generated at a fixed time each day.

The schedule runs at `00:05` instead of exactly midnight, providing a five-minute buffer for the previous day's final records to reach DynamoDB.

## 5.15.2. Configuration overview

| Component | Value |
|---|---|
| Schedule name | `SmartHomeDailyExport` |
| Schedule group | `default` |
| Schedule type | Recurring |
| Expression | `cron(5 0 * * ? *)` |
| Time zone | `Asia/Ho_Chi_Minh` |
| Flexible time window | Off |
| Target | `SmartHomeExport` |
| Payload | `{}` |
| Execution role | `SmartHomeSchedulerRole` |
| Maximum event age | 1 hour |
| Maximum retries | 2 |
| Dead-letter queue | None |
| State | Enabled |

## 5.15.3. The two execution roles

The project uses two different roles:

| Role | Used by | Purpose |
|---|---|---|
| `SmartHomeExportRole` | Lambda | Reads DynamoDB, writes S3, and writes CloudWatch Logs |
| `SmartHomeSchedulerRole` | EventBridge Scheduler | Invokes `SmartHomeExport` |

Scheduler does not use `SmartHomeExportRole` to invoke Lambda. Each service uses a role that matches its responsibility.

## 5.15.4. Create the Scheduler execution role

Perform this step using the root or administrator account.

Navigate to:

```text
IAM
→ Roles
→ Create role
→ Custom trust policy
```

Enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "scheduler.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "<ACCOUNT_ID>",
          "aws:SourceArn": "arn:aws:scheduler:ap-southeast-1:<ACCOUNT_ID>:schedule-group/default"
        }
      }
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with the actual AWS account ID.

The `aws:SourceAccount` and `aws:SourceArn` conditions restrict the role to the correct account and the `default` schedule group.

Choose **Next** without selecting a managed policy.

Enter:

```text
SmartHomeSchedulerRole
```

Choose:

```text
Create role
```

## 5.15.5. Allow Scheduler to invoke Lambda

Open:

```text
IAM
→ Roles
→ SmartHomeSchedulerRole
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
      "Sid": "InvokeSmartHomeExport",
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": [
        "arn:aws:lambda:ap-southeast-1:<ACCOUNT_ID>:function:SmartHomeExport",
        "arn:aws:lambda:ap-southeast-1:<ACCOUNT_ID>:function:SmartHomeExport:*"
      ]
    }
  ]
}
```

Name the policy:

```text
SmartHomeSchedulerInvokeLambda
```

Choose:

```text
Create policy
```

The role can invoke only `SmartHomeExport`, not every Lambda function in the account.

## 5.15.6. Grant Scheduler permissions to the IAM user

The `dinh-fcj` IAM user needs permission to create the schedule and pass `SmartHomeSchedulerRole` to Scheduler.

Using the root or administrator account, navigate to:

```text
IAM
→ Policies
→ Create policy
→ JSON
```

Enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ViewSchedulerConsole",
      "Effect": "Allow",
      "Action": [
        "scheduler:ListSchedules",
        "scheduler:ListScheduleGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ViewDefaultScheduleGroup",
      "Effect": "Allow",
      "Action": [
        "scheduler:GetScheduleGroup"
      ],
      "Resource": "arn:aws:scheduler:ap-southeast-1:<ACCOUNT_ID>:schedule-group/default"
    },
    {
      "Sid": "ManageSmartHomeDailyExport",
      "Effect": "Allow",
      "Action": [
        "scheduler:CreateSchedule",
        "scheduler:GetSchedule",
        "scheduler:UpdateSchedule",
        "scheduler:ListTagsForResource",
        "scheduler:TagResource",
        "scheduler:UntagResource"
      ],
      "Resource": "arn:aws:scheduler:ap-southeast-1:<ACCOUNT_ID>:schedule/default/SmartHomeDailyExport"
    },
    {
      "Sid": "PassOnlySchedulerRole",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/SmartHomeSchedulerRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "scheduler.amazonaws.com"
        }
      }
    },
    {
      "Sid": "ViewSchedulerRole",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:ListRoles"
      ],
      "Resource": "*"
    }
  ]
}
```

Name the policy:

```text
SmartHomeSchedulerDeveloperPolicy
```

Choose:

```text
Create policy
```

Attach it to:

```text
IAM
→ User groups
→ SmartHome-Developers
→ Permissions
→ Add permissions
→ Attach policies
```

Select:

```text
SmartHomeSchedulerDeveloperPolicy
```

Sign out of the root account and sign in as:

```text
dinh-fcj
```

## 5.15.7. Create the daily schedule

Confirm the Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Navigate to:

```text
Amazon EventBridge
→ Scheduler
→ Schedules
→ Create schedule
```

Enter:

| Property | Value |
|---|---|
| Schedule name | `SmartHomeDailyExport` |
| Schedule group | `default` |
| Description | `Export previous day smart home report to S3` |
| Occurrence | `Recurring schedule` |

Select:

```text
Cron-based schedule
```

Enter:

```text
cron(5 0 * * ? *)
```

If the console displays six separate fields, use:

| Field | Value |
|---|---|
| Minutes | `5` |
| Hours | `0` |
| Day of month | `*` |
| Month | `*` |
| Day of week | `?` |
| Year | `*` |

Select the time zone:

```text
Asia/Ho_Chi_Minh
```

Do not select UTC. A schedule at `00:05 UTC` would run at `07:05` in Vietnam.

Set:

```text
Flexible time window: Off
```

Leave the start and end dates empty.

Choose:

```text
Next
```

## 5.15.8. Select the Lambda target

On the **Select target** page, choose:

```text
Templated targets
→ AWS Lambda
→ Invoke
```

Select:

```text
SmartHomeExport
```

Leave the version or alias empty to invoke the current function version.

For **Input**, enter exactly:

```json
{}
```

Do not provide `reportDate`. With an empty event, Lambda automatically selects the previous day in `Asia/Ho_Chi_Minh`.

Choose:

```text
Next
```

## 5.15.9. Configure retry, encryption, and permissions

Configure:

| Property | Value |
|---|---|
| Schedule state | Enable schedule |
| Action after completion | None |
| Retry policy | Enabled |
| Maximum age of event | 1 hour |
| Maximum retry attempts | 2 |
| Dead-letter queue | None |

Under **Encryption**, do not select:

```text
Customize encryption settings (advanced)
```

Without custom encryption settings, Scheduler uses AWS-managed default encryption. No customer-managed KMS key or additional KMS permission is required.

Under **Permissions**, select:

```text
Use existing role
```

Choose:

```text
SmartHomeSchedulerRole
```

Do not select **Create new role**, because the IAM user is only allowed to pass the role prepared by the administrator.

Choose:

```text
Next
```

![Schedule target and execution role](/images/5.15.9.png)



## 5.15.10. Review and create the schedule

Verify:

| Property | Value |
|---|---|
| Name | `SmartHomeDailyExport` |
| Group | `default` |
| State | Enabled |
| Expression | `cron(5 0 * * ? *)` |
| Time zone | `Asia/Ho_Chi_Minh` |
| Flexible window | Off |
| Target | `SmartHomeExport` |
| Input | `{}` |
| Execution role | `SmartHomeSchedulerRole` |
| Retry attempts | 2 |
| Maximum event age | 1 hour |

Choose:

```text
Create schedule
```

Open the new schedule and verify:

```text
State: Enabled
```

The **Next invocation time** must correspond to `00:05` on the next day in Vietnam time.

![SmartHomeDailyExport details](/images/5.15.10.png)



## 5.15.11. How the report date is selected

Assume Scheduler runs at:

```text
2026-09-15 00:05 Asia/Ho_Chi_Minh
```

It sends:

```json
{}
```

Lambda calculates the previous date:

```text
2026-09-14
```

It then creates:

```text
reports/home01/esp32-01/2026/09/14/report.json
reports/home01/esp32-01/2026/09/14/report.csv
```

The report date is calculated by Lambda rather than being hard-coded in the schedule.

## 5.15.12. Verify the automatic invocation

After `00:05`, wait approximately one to three minutes and navigate to:

```text
Amazon S3
→ smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
→ reports
→ home01
→ esp32-01
→ YYYY
→ MM
→ DD
```

Verify that:

- `report.json` exists.
- `report.csv` exists.
- **Last modified** is later than the scheduled invocation.
- Both files are larger than `0 B`.

![Report generated by Scheduler](/images/5.15.12.png)



Also check:

```text
AWS Lambda
→ SmartHomeExport
→ Monitor
```

Expected metrics:

| Metric | Expected result |
|---|---|
| Invocations | Increased by 1 |
| Errors | 0 |
| Throttles | 0 |
| Duration | Below the 30-second timeout |

## 5.15.13. Effect of S3 Versioning

If a report already exists from a manual Lambda test, Scheduler writes to the same object key.

Because Versioning is enabled:

- The old report is preserved.
- The scheduled report becomes the current version.
- The manually generated report becomes a noncurrent version.

Use **Show versions** to inspect previous versions. No additional screenshot is required.

## 5.15.14. Common issues

### SmartHomeSchedulerRole cannot be selected

Verify that:

- The role exists.
- Its trust policy allows `scheduler.amazonaws.com`.
- The IAM user has `iam:GetRole`.
- The IAM user has `iam:PassRole`.
- `iam:PassedToService` is `scheduler.amazonaws.com`.

### The console displays kms:ListAliases

Clear:

```text
Customize encryption settings (advanced)
```

This workshop does not use a customer-managed KMS key.

### The schedule is enabled, but Lambda is not invoked

Verify that:

- The target is `SmartHomeExport`.
- The execution role is `SmartHomeSchedulerRole`.
- The role has `lambda:InvokeFunction`.
- The Lambda ARN contains the correct Region and account.
- The next invocation time has passed.

### The schedule runs at the wrong time

Verify:

```text
Time zone: Asia/Ho_Chi_Minh
```

Do not use UTC for this workshop schedule.

### Lambda runs, but no S3 report appears

Open:

```text
/aws/lambda/SmartHomeExport
```

Search for:

```text
AccessDeniedException
ResourceNotFoundException
Task timed out
```

Also verify:

```text
TABLE_NAME
REPORT_BUCKET
DEVICE_KEY
HOME_ID
DEVICE_ID
```

### The report contains zero records

The previous day might not contain data. The schedule and Lambda can still complete successfully.

Verify the selected report date, `DEVICE_KEY`, and DynamoDB timestamps.

## 5.15.15. Result

After completing this section:

- `SmartHomeSchedulerRole` has been created.
- The role can invoke only `SmartHomeExport`.
- The IAM user can manage the specific schedule and pass the correct role.
- `SmartHomeDailyExport` runs at 00:05 Vietnam time.
- Scheduler sends `{}` to Lambda.
- Lambda exports the previous day's records.
- JSON and CSV reports are stored in S3.
- S3 Versioning preserves previous report versions.
- Daily report generation is fully automated.

## References

- [Getting started with EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/getting-started.html)
- [Schedule types in EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html)
- [Using templated targets](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-templated.html)
- [Confused deputy prevention](https://docs.aws.amazon.com/scheduler/latest/UserGuide/cross-service-confused-deputy-prevention.html)