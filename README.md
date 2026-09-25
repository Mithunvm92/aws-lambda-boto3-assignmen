# Assignment 2 – S3 Log Cleaner

## Objective

Create an AWS Lambda function using Python and Boto3 to automatically identify and delete S3 log files that are older than 90 days.

The Lambda function will:

- Connect to an Amazon S3 bucket.
- List objects stored in the specified log location.
- Check the `LastModified` date of each object.
- Identify log files older than 90 days.
- Delete objects that exceed the retention period.
- Record the cleanup activity in CloudWatch Logs.

---

## AWS Services Used

- AWS Lambda
- Amazon S3
- AWS IAM
- Amazon CloudWatch Logs
- Amazon EventBridge

---

## Architecture

```text
                    Amazon S3
                       |
                       | Log Files
                       v
                 AWS Lambda
                       |
              List S3 Objects
                       |
                       v
              Check LastModified
                       |
              +--------+--------+
              |                 |
          < 90 Days          > 90 Days
              |                 |
             Keep              Delete
                                |
                                v
                       CloudWatch Logs
```

---

## Prerequisites

- AWS account
- Permission to create Lambda functions
- Permission to create IAM roles and policies
- An S3 bucket containing log files
- Python 3.x runtime supported by AWS Lambda

---

# Step 1 – Create S3 Bucket

An S3 bucket is created for testing the log-cleaning Lambda function.

Example bucket name:

```text
lambda-s3-log-cleaner-<unique-name>
```

A `logs/` prefix is used to store log files.

Example structure:

```text
lambda-s3-log-cleaner-<unique-name>/
└── logs/
    └── test-log.txt
```

The bucket is used as the source for the Lambda cleanup operation.

> **Note:** For testing, the Lambda function checks the actual `LastModified` timestamp of S3 objects. The test environment should contain objects that can demonstrate the retention logic.

---

# Step 2 – Create IAM Role

An IAM execution role is created for the Lambda function.

Example role:

```text
s3-log-cleaner-lambda-role
```

The role allows Lambda to:

- List objects in the S3 bucket.
- Read S3 objects when required.
- Delete objects that exceed the retention period.
- Write execution logs to CloudWatch Logs.

---

## IAM Policy

The following policy provides the required S3 permissions.

Replace `YOUR_BUCKET_NAME` with the actual S3 bucket name.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME"
    },
    {
      "Sid": "DeleteOldLogs",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/logs/*"
    }
  ]
}
```

### Permission Description

| Permission | Purpose |
|---|---|
| `s3:ListBucket` | Allows Lambda to list objects in the S3 bucket |
| `s3:GetObject` | Allows Lambda to access objects when required |
| `s3:DeleteObject` | Allows Lambda to delete old log files |

---

# Step 3 – Create Lambda Function

A Lambda function is created with the following configuration:

| Configuration | Value |
|---|---|
| Function Name | `s3-log-cleaner` |
| Runtime | Python 3.x |
| Architecture | x86_64 |
| Execution Role | `s3-log-cleaner-lambda-role` |

---

# Step 4 – Lambda Function Code

The Lambda function uses Boto3 to list S3 objects and delete objects older than 90 days.

The bucket name and log prefix are configured in the function.

```python
import boto3
from datetime import datetime, timezone, timedelta
from botocore.exceptions import ClientError


s3 = boto3.client("s3")

BUCKET_NAME = "YOUR_BUCKET_NAME"
LOG_PREFIX = "logs/"
RETENTION_DAYS = 90


def lambda_handler(event, context):

    cutoff_date = datetime.now(timezone.utc) - timedelta(
        days=RETENTION_DAYS
    )

    deleted_objects = []
    retained_objects = []

    print("===== S3 LOG CLEANUP STARTED =====")
    print(f"Bucket: {BUCKET_NAME}")
    print(f"Prefix: {LOG_PREFIX}")
    print(f"Retention period: {RETENTION_DAYS} days")
    print(f"Cutoff date: {cutoff_date.isoformat()}")

    try:
        paginator = s3.get_paginator("list_objects_v2")

        for page in paginator.paginate(
            Bucket=BUCKET_NAME,
            Prefix=LOG_PREFIX
        ):

            for obj in page.get("Contents", []):

                object_key = obj["Key"]
                last_modified = obj["LastModified"]

                if last_modified < cutoff_date:

                    s3.delete_object(
                        Bucket=BUCKET_NAME,
                        Key=object_key
                    )

                    deleted_objects.append(object_key)

                    print(
                        f"DELETED: {object_key} "
                        f"(LastModified: {last_modified.isoformat()})"
                    )

                else:

                    retained_objects.append(object_key)

                    print(
                        f"RETAINED: {object_key} "
                        f"(LastModified: {last_modified.isoformat()})"
                    )

        print("\n===== S3 LOG CLEANUP REPORT =====")
        print(f"Deleted objects: {len(deleted_objects)}")
        print(f"Retained objects: {len(retained_objects)}")

        return {
            "statusCode": 200,
            "deleted_objects": deleted_objects,
            "retained_objects": retained_objects
        }

    except ClientError as error:

        print(
            f"S3 operation failed: "
            f"{error.response['Error']['Code']}"
        )

        raise
```

---

# Step 5 – Configure Bucket Name

Replace:

```text
YOUR_BUCKET_NAME
```

with the actual S3 bucket name.

For example:

```python
BUCKET_NAME = "lambda-s3-log-cleaner-mithun-2026"
```

Keep:

```python
LOG_PREFIX = "logs/"
```

unless a different S3 prefix is being used.

---

# Step 6 – Test Configuration

A Lambda test event is created for manual testing.

### Test Event Name

```text
test-s3-log-cleaner
```

### Test Event

```json
{}
```

The Lambda function is manually invoked using this test event.

---

# Step 7 – Expected Execution

During execution, the Lambda function checks every object under the configured `logs/` prefix.

Objects older than 90 days are deleted.

Objects newer than 90 days are retained.

Example CloudWatch output:

```text
===== S3 LOG CLEANUP STARTED =====
Bucket: lambda-s3-log-cleaner-example
Prefix: logs/
Retention period: 90 days

DELETED: logs/old-log.txt
RETAINED: logs/recent-log.txt

===== S3 LOG CLEANUP REPORT =====
Deleted objects: 1
Retained objects: 1
```

---

# Step 8 – CloudWatch Logs

AWS Lambda automatically sends the function's output to Amazon CloudWatch Logs.

The logs can be viewed from:

```text
AWS Console
    ↓
Lambda
    ↓
s3-log-cleaner
    ↓
Monitor
    ↓
View CloudWatch logs
```

The CloudWatch logs provide evidence of:

- Lambda execution
- Objects checked
- Objects deleted
- Objects retained
- Cleanup summary

---

# Step 9 – Schedule the Cleanup

The cleanup can be automated using Amazon EventBridge.

A scheduled EventBridge rule can invoke the Lambda function periodically.

Example schedule:

```text
Once per week
```

The EventBridge rule invokes:

```text
s3-log-cleaner
```

This allows old S3 log files to be automatically cleaned without manual execution.

---

# Testing

The Lambda function is tested by manually invoking it from the AWS Lambda console.

## Test Input

```json
{}
```

## Expected Behavior

The function should:

1. List objects under the configured S3 log prefix.
2. Calculate the 90-day retention cutoff.
3. Compare each object's `LastModified` timestamp with the cutoff.
4. Delete objects older than 90 days.
5. Retain objects newer than 90 days.
6. Write the cleanup results to CloudWatch Logs.
7. Return a successful Lambda response.

---

## Expected Result

```text
Lambda execution: SUCCESS
```

Example response:

```json
{
  "statusCode": 200,
  "deleted_objects": [
    "logs/old-log.txt"
  ],
  "retained_objects": [
    "logs/recent-log.txt"
  ]
}
```

---

# Screenshots

The following screenshots should be included as evidence for the assignment.

## 1. S3 Bucket

Show the S3 bucket and `logs/` folder.

Save as:

```text
screenshots/01-s3-bucket.png
```

---

## 2. IAM Role and Policy

Show the Lambda execution role and S3 permissions.

Save as:

```text
screenshots/02-iam-policy.png
```

---

## 3. Lambda Configuration

Show the Lambda function name, Python runtime and execution role.

Save as:

```text
screenshots/03-lambda-configuration.png
```

---

## 4. Lambda Source Code

Show the deployed Lambda Python code.

Save as:

```text
screenshots/04-lambda-code.png
```

---

## 5. Lambda Test Success

Show the successful Lambda test execution and response.

Save as:

```text
screenshots/05-lambda-test-success.png
```

---

## 6. CloudWatch Logs

Show the cleanup report containing deleted and retained objects.

Save as:

```text
screenshots/06-cloudwatch-logs.png
```

---

## 7. S3 Verification

Show the S3 bucket after the cleanup operation to demonstrate that the targeted old object was removed.

Save as:

```text
screenshots/07-s3-cleanup-result.png
```

---

# Project Structure

```text
assignment-02-s3-log-cleaner/
│
├── lambda_function.py
├── README.md
│
└── screenshots/
    ├── 01-s3-bucket.png
    ├── 02-iam-policy.png
    ├── 03-lambda-configuration.png
    ├── 04-lambda-code.png
    ├── 05-lambda-test-success.png
    ├── 06-cloudwatch-logs.png
    └── 07-s3-cleanup-result.png
```

---

# Conclusion

This assignment demonstrates the use of AWS Lambda and Boto3 to automate S3 log cleanup.

The solution identifies objects older than the configured 90-day retention period, deletes those objects, retains newer objects, and records the cleanup activity in CloudWatch Logs.

The Lambda function can also be scheduled using Amazon EventBridge for automated periodic cleanup.
