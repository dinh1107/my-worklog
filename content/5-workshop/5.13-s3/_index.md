---
title: "Creating an S3 Report Bucket"
date: 2026-09-14
weight: 13
chapter: false
pre: "<b>5.13. </b>"
---

# Creating an Amazon S3 Bucket for Reports

DynamoDB stores individual telemetry and event records. However, it is not intended to store complete report files for downloading or sharing.

In this section, we will create an Amazon S3 bucket for daily reports in two formats:

```text
JSON
CSV
```

The report export Lambda function will be created in the next section.

## 5.13.1. Role of Amazon S3

Amazon Simple Storage Service — Amazon S3 — is an object storage service. Each file is stored as an object inside a bucket.

This project uses:

| Service | Stored data |
|---|---|
| DynamoDB | Individual telemetry and event records |
| Amazon S3 | Aggregated JSON and CSV report files |

The planned export flow is:

```text
DynamoDB
→ SmartHomeExport
→ Amazon S3
```

Amazon S3 is used because it:

- Is designed for file storage.
- Allows reports to be downloaded.
- Can retain multiple versions of the same file.
- Encrypts stored objects.
- Supports IAM access control.
- Does not require a dedicated storage server.

## 5.13.2. Bucket naming convention

An S3 bucket name must be unique within an AWS partition. The AWS account ID and Region are therefore included in the name.

Use:

```text
smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

Example:

```text
smarthome-reports-123456789012-ap-southeast-1
```

Replace `<ACCOUNT_ID>` with your AWS account ID.

Do not include an email address, password, personal name, or other sensitive information in the bucket name because it can appear in object ARNs and URLs.

## 5.13.3. Configuration overview

| Property | Value |
|---|---|
| Bucket type | General purpose |
| Bucket name | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |
| Region | `ap-southeast-1` |
| Object Ownership | Bucket owner enforced |
| ACL | Disabled |
| Block Public Access | All enabled |
| Bucket Versioning | Enabled |
| Default encryption | SSE-S3 |
| Object Lock | Disabled |

Create the bucket using the root or administrator account.

## 5.13.4. Create the S3 bucket

### Step 1: Open Amazon S3

In the AWS Management Console, search for:

```text
Amazon S3
```

Navigate to:

```text
Amazon S3
→ General purpose buckets
→ Create bucket
```

### Step 2: Select the Region and bucket name

Under **General configuration**, enter:

| Property | Value |
|---|---|
| AWS Region | `Asia Pacific (Singapore) ap-southeast-1` |
| Bucket type | `General purpose` |
| Bucket name | `smarthome-reports-<ACCOUNT_ID>-ap-southeast-1` |

Do not copy settings from another bucket.

If AWS displays:

```text
Bucket with the same name already exists
```

verify the account ID or add a unique suffix to the bucket name.

### Step 3: Configure Object Ownership

Under **Object Ownership**, select:

```text
ACLs disabled
Bucket owner enforced
```

With this configuration:

- ACLs are not used.
- Access is controlled using IAM and bucket policies.
- The AWS account that owns the bucket also owns uploaded objects.

This is the default and recommended configuration for the workshop.

### Step 4: Block public access

Under **Block Public Access settings for this bucket**, keep:

```text
Block all public access
```

Verify that all four options are enabled.

Smart home reports can contain device activity times, door states, and sensor data. Therefore, the bucket must not be public.

Do not enable S3 static website hosting for this bucket.

### Step 5: Enable Bucket Versioning

Under **Bucket Versioning**, select:

```text
Enable
```

The export function uses a fixed report path for each date. If a report is generated again, Versioning preserves the previous object instead of permanently replacing it.

For example, the following object key can have multiple versions:

```text
reports/home01/esp32-01/2026/09/14/report.json
```

### Step 6: Configure encryption

Under **Default encryption**, select:

```text
Server-side encryption with Amazon S3 managed keys
SSE-S3
```

A customer-managed KMS key is not required for this workshop.

SSE-S3 encrypts stored data without requiring an additional KMS key or KMS permissions for the Lambda function.

### Step 7: Review the remaining settings

Keep:

| Property | Value |
|---|---|
| Tags | Optional |
| Object Lock | Disabled |
| Advanced settings | Default |

Optional project tags:

| Key | Value |
|---|---|
| `Project` | `SmartHome` |
| `Environment` | `Workshop` |

Choose:

```text
Create bucket
```

![Creating the report bucket](/images/5.13.4.png)


## 5.13.5. Verify the bucket

Open the bucket:

```text
Amazon S3
→ General purpose buckets
→ smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

Open the **Properties** tab and verify:

| Property | Expected result |
|---|---|
| AWS Region | Asia Pacific (Singapore) |
| Bucket Versioning | Enabled |
| Default encryption | SSE-S3 |
| Object Lock | Disabled |

Open the **Permissions** tab and verify:

```text
Block all public access: On
```

![S3 bucket properties](/images/5.13.5.png)



## 5.13.6. Grant bucket read access to the IAM user

The bucket was created using the root or administrator account. To allow `dinh-fcj` to inspect and download reports, create a policy limited to this bucket.

Navigate to:

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
      "Sid": "ListS3BucketsInConsole",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ViewSmartHomeReportsBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1"
    },
    {
      "Sid": "ReadSmartHomeReports",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::smarthome-reports-<ACCOUNT_ID>-ap-southeast-1/*"
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with the actual AWS account ID.

Name the policy:

```text
SmartHomeReportsDeveloperPolicy
```

Choose:

```text
Create policy
```

Attach the policy to:

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
SmartHomeReportsDeveloperPolicy
```

The policy allows the IAM user to:

- View the bucket in the AWS Console.
- View its Region and Versioning configuration.
- List report objects.
- Open and download reports.
- View object versions.

It does not allow the IAM user to:

- Make the bucket public.
- Delete the bucket.
- Disable Versioning.
- Delete reports.
- Modify the bucket policy.

## 5.13.7. IAM user permissions compared with Lambda permissions

The policy created above applies only to the `dinh-fcj` IAM user.

The `SmartHomeExport` function will use a separate execution role with:

```text
s3:PutObject
```

That permission will be configured when creating the export Lambda function.

| Identity | Purpose |
|---|---|
| IAM user `dinh-fcj` | Views and downloads reports |
| `SmartHomeExportRole` | Allows Lambda to create report files |
| S3 bucket policy | Controls bucket-level access when required |

Do not grant `s3:*` to the entire bucket when the function only needs limited object access.

## 5.13.8. Report object structure

The export function will create objects using:

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

Example for September 14, 2026:

```text
reports/home01/esp32-01/2026/09/14/report.json
reports/home01/esp32-01/2026/09/14/report.csv
```

The path components mean:

| Component | Purpose |
|---|---|
| `reports/` | Root report prefix |
| `home01/` | Groups reports by home |
| `esp32-01/` | Groups reports by device |
| `2026/09/14/` | Groups reports by date |
| `report.json` | Structured report for applications |
| `report.csv` | Tabular report for spreadsheet software |

S3 folders are object key prefixes rather than physical directories. The folders do not need to be created manually; Lambda creates the key structure when uploading each object.

## 5.13.9. Why enable Versioning?

If `SmartHomeExport` runs several times for the same date, it writes to the same object key.

With Versioning enabled:

- The previous object remains available.
- The latest object becomes the current version.
- An earlier report can be recovered.
- Re-running Lambda does not permanently destroy the previous report.

Example:

```text
report.json
├── Version 1: manual test
├── Version 2: Lambda test after a code update
└── Version 3: automatic Scheduler invocation
```

Versioning can increase storage usage. A production environment should use a lifecycle rule to remove noncurrent versions after an appropriate period.

A lifecycle rule is not required for this workshop.

## 5.13.10. Verify access as the IAM user

Sign out of the root or administrator account and sign in as:

```text
dinh-fcj
```

Open Amazon S3 and select the report bucket.

Expected result:

- The IAM user can see the bucket.
- The Objects tab can be opened.
- Basic bucket properties can be viewed.
- Public access settings and bucket deletion are not allowed.
- The Objects tab can still be empty because the export function has not been created.

An empty bucket is expected at this stage.

## 5.13.11. Common issues

### The bucket name already exists

S3 bucket names must be unique. Use:

```text
smarthome-reports-<ACCOUNT_ID>-ap-southeast-1
```

or append another unique suffix.

### The bucket was created in the wrong Region

The bucket Region cannot be changed after creation. Use:

```text
ap-southeast-1
```

to match Lambda and DynamoDB.

### The IAM user cannot see the bucket

Verify that:

- `SmartHomeReportsDeveloperPolicy` exists.
- The policy is attached to `SmartHome-Developers`.
- `dinh-fcj` belongs to the correct group.
- The policy contains `s3:ListAllMyBuckets` and `s3:ListBucket`.

### The bucket opens but contains no objects

This is expected at this stage. Report objects appear only after `SmartHomeExport` is created and invoked.

### The bucket is shown as Public

Do not continue while the bucket is public. Open **Permissions** and enable:

```text
Block all public access
```

Do not add a bucket policy with `"Principal": "*"`.

## 5.13.12. Result

After completing this section:

- The report bucket has been created in Singapore.
- Block Public Access is enabled.
- ACLs are disabled.
- Bucket Versioning is enabled.
- Data is encrypted with SSE-S3.
- The IAM user can view and download reports.
- The JSON and CSV object-key structure has been defined.
- The bucket is ready to receive objects from `SmartHomeExport`.

## References

- [Getting started with Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/GetStartedWithS3.html#creating-bucket)
- [Using S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [Blocking public access to S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Amazon S3 default encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-encryption-faq.html)