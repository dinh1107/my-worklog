---
title: "Creating the IAM Execution Role for the Ingest Lambda"
weight: 7
chapter: false
pre: "<b>5.7. </b>"
---

# Creating the IAM Execution Role for the Ingest Lambda

In this section, an IAM execution role is created for the `SmartHomeIngest` Lambda function. The role allows the function to write execution logs to Amazon CloudWatch Logs and store data in the `SmartHomeTelemetry` DynamoDB table.

This section is completed using the root or administrator account because the `dinh-fcj` IAM user does not have permission to create roles or manage IAM policies.

## 5.7.1. Objectives and Permission Separation

After completing this section, the following components are available:

| Component | Name | Attached to |
|---|---|---|
| Lambda execution role | `SmartHomeIngestRole` | `SmartHomeIngest` Lambda |
| AWS managed policy | `AWSLambdaBasicExecutionRole` | `SmartHomeIngestRole` |
| Inline policy | `SmartHomeIngestDynamoDBWrite` | `SmartHomeIngestRole` |
| Developer policy | `SmartHomeLambdaDeveloperPolicy` | `SmartHome-Developers` group |

The two permission types have different purposes:

- The **execution role** contains permissions used by Lambda while the function is running.
- The **developer policy** allows the IAM user to create Lambda functions and pass the execution role to Lambda.

Do not attach `SmartHomeLambdaDeveloperPolicy` to the execution role, and do not attach `SmartHomeIngestDynamoDBWrite` to the IAM user.

## 5.7.2. Create SmartHomeIngestRole

Sign in using the root or administrator account, and then open:

```text
IAM → Roles → Create role
```

For **Trusted entity type**, select:

```text
AWS service
```

For **Service or use case**, select:

```text
Lambda
```

Select **Next** to continue to the permissions step.

Selecting Lambda as the trusted service permits `lambda.amazonaws.com` to use the role through the `sts:AssumeRole` action.

AWS creates the following trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

![Select AWS Lambda as the trusted service](/images/5.7.2.png)

## 5.7.3. Attach CloudWatch Logs Permissions

In the permission policies list, search for and select:

```text
AWSLambdaBasicExecutionRole
```

This AWS managed policy allows Lambda to:

- Create a CloudWatch log group.
- Create log streams.
- Write log events during function execution.

Select **Next**, and enter the role name:

```text
SmartHomeIngestRole
```

Verify that the trusted entity is Lambda and that `AWSLambdaBasicExecutionRole` is selected. Then choose:

```text
Create role
```

![Create SmartHomeIngestRole](/images/5.7.3.png)

## 5.7.4. Grant DynamoDB Write Permission

The new role can write CloudWatch Logs but cannot yet write data to DynamoDB. The Lambda function requires permission to add items to the `SmartHomeTelemetry` table.

Open:

```text
IAM → Roles → SmartHomeIngestRole
→ Permissions → Add permissions
→ Create inline policy
```

Select the **JSON** tab and enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteSmartHomeTelemetry",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:ap-southeast-1:079755087993:table/SmartHomeTelemetry"
    }
  ]
}
```

Select **Next** and enter the policy name:

```text
SmartHomeIngestDynamoDBWrite
```

Then select **Create policy**.

The policy permits only `dynamodb:PutItem` on the `SmartHomeTelemetry` table. `AmazonDynamoDBFullAccess` is not used because the function does not require administrative access to other DynamoDB tables.

![SmartHomeIngestDynamoDBWrite inline policy](/images/5.7.3.png)


## 5.7.5. Verify the IAM User's Deployment Permissions

The `SmartHomeIngest` Lambda function will be created using the `dinh-fcj` IAM user. During the project implementation, the root account created the following policy:

```text
SmartHomeLambdaDeveloperPolicy
```

and attached it to:

```text
SmartHome-Developers
```

The policy includes:

- Permission to create, view, update, and test the `SmartHomeIngest` Lambda function.
- Permission to view `SmartHomeIngestRole` in the Lambda console.
- `iam:PassRole` permission for `SmartHomeIngestRole`.
- A condition that permits the role to be passed only to `lambda.amazonaws.com`.

Open:

```text
IAM → User groups → SmartHome-Developers → Permissions
```

Verify that `SmartHomeLambdaDeveloperPolicy` is attached to the group.

![SmartHomeLambdaDeveloperPolicy attached to the group](/images/5.7.5.png)


## 5.7.6. Verify the Result

Open:

```text
IAM → Roles → SmartHomeIngestRole
```

Verify the following values:

| Setting | Expected result |
|---|---|
| Trusted service | `lambda.amazonaws.com` |
| Managed policy | `AWSLambdaBasicExecutionRole` |
| Inline policy | `SmartHomeIngestDynamoDBWrite` |
| DynamoDB action | `dynamodb:PutItem` |
| DynamoDB resource | `SmartHomeTelemetry` table |
| SNS permission | Not added in this section |

Next, verify that the `SmartHome-Developers` group contains `SmartHomeLambdaDeveloperPolicy`.

When all settings are correct, sign out of the root account and sign back in as:

```text
dinh-fcj
```

This IAM user will use `SmartHomeIngestRole` when creating the `SmartHomeIngest` Lambda function in the next section.
