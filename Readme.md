# AWS Serverless DynamoDB Backup to S3

A hands-on AWS project demonstrating an end-to-end serverless data pipeline using **AWS Lambda, Amazon DynamoDB, Amazon S3, Amazon EventBridge Scheduler, Python, and Boto3**.

This project was completed as part of my **BongoDev DevOps Bootcamp** to gain practical experience with AWS serverless computing, NoSQL databases, IAM permissions, automated scheduling, cloud monitoring, and data export workflows.

## Project Overview

The objective of this project was to retrieve sample user information from a public REST API, store the data in Amazon DynamoDB, and automatically export the stored records as timestamped JSON files to Amazon S3.

The project consists of two AWS Lambda functions and Amazon EventBridge Scheduler.

**Main Components:**

- **Lambda #1 – Import Users:** Retrieves 10 sample user records from the JSONPlaceholder API and stores them in DynamoDB.
- **Lambda #2 – Backup to S3:** Scans the DynamoDB table, converts the retrieved records into JSON format, and uploads timestamped backup files to Amazon S3.
- **Amazon EventBridge Scheduler:** Automatically invokes Lambda #2 every 2 minutes.
- **Amazon CloudWatch:** Collects execution logs and metrics from both Lambda functions.

The first Lambda function is invoked manually, while the second Lambda function can be invoked manually for testing and automatically through EventBridge Scheduler.

## Architecture

```text
                 JSONPlaceholder API
                 (10 Sample Users)
                          |
                          | HTTP GET
                          v
              AWS Lambda #1: Import Users
              import-jsonplaceholder-users
                          |
                          | boto3: dynamodb:PutItem
                          v
                  Amazon DynamoDB
                 jsonplaceholder-users
                          |
                          | boto3: dynamodb:Scan
                          v
              AWS Lambda #2: Backup to S3
                 export-dynamodb-to-s3
                          |
                          | boto3: s3:PutObject
                          v
                  Private Amazon S3
                   JSON Backup Files
                          ^
                          |
                          | Scheduled Invocation
                          |
               Amazon EventBridge Scheduler
                     rate(2 minutes)


          BOTH Lambda Functions
                    |
                    v
             Amazon CloudWatch
               Logs / Metrics
```

### Workflow Explanation

1. Lambda #1 sends an HTTP GET request to the JSONPlaceholder API.
2. The API returns 10 sample user records.
3. Lambda #1 uses Boto3 and `dynamodb:PutItem` to insert the records into the DynamoDB table.
4. EventBridge Scheduler invokes Lambda #2 according to the configured schedule.
5. Lambda #2 uses `dynamodb:Scan` to retrieve records from DynamoDB.
6. The retrieved records are converted into JSON-compatible data.
7. Lambda #2 uses `s3:PutObject` to upload the JSON backup file to the private S3 bucket.
8. Every successful backup execution creates a separate timestamped JSON file.
9. Both Lambda functions send execution logs and metrics to Amazon CloudWatch.

**Important:** EventBridge Scheduler does not directly upload files to S3. It invokes Lambda #2, which performs the actual DynamoDB-to-S3 export operation.

## Technologies Used

| Technology | Purpose |
|---|---|
| AWS Lambda | Execute serverless Python functions |
| Amazon DynamoDB | Store user information |
| Amazon S3 | Store timestamped JSON backup files |
| Amazon EventBridge Scheduler | Automatically invoke Lambda #2 every 2 minutes |
| AWS IAM | Manage service permissions and execution roles |
| Amazon CloudWatch | Monitor Lambda logs and metrics |
| Python | Implement Lambda functions |
| Boto3 | Interact with AWS services using Python |
| JSONPlaceholder API | Provide sample user data |
| Git & GitHub | Version control and project documentation |

## AWS Configuration

| Resource | Configuration |
|---|---|
| AWS Region | `us-west-2` (US West – Oregon) |
| DynamoDB Table | `jsonplaceholder-users` |
| Partition Key | `id` |
| Partition Key Type | Number |
| Sort Key | None |
| S3 Bucket | `student-lambda-backups-createdbytomal` |
| S3 Backup Prefix | `backups/` |
| Import Lambda | `import-jsonplaceholder-users` |
| Backup Lambda | `export-dynamodb-to-s3` |
| EventBridge Schedule | `dynamodb-backup-every-2min` |
| Schedule Expression | `rate(2 minutes)` |
| Schedule Group | `default` |

## Implementation Steps

### 1. Amazon DynamoDB Setup

Created an Amazon DynamoDB table named `jsonplaceholder-users`.

**Configuration:**

- Table name: `jsonplaceholder-users`
- Partition key: `id`
- Partition key type: Number
- Sort key: None

The table stores user information retrieved from the JSONPlaceholder API.

The `id` attribute uniquely identifies each user record.

### 2. Amazon S3 Bucket Setup

Created an Amazon S3 bucket named `student-lambda-backups-createdbytomal` to store exported JSON backup files.

**Configuration:**

- AWS Region: `us-west-2`
- Bucket access: Private
- Block Public Access: Enabled
- Default server-side encryption: Enabled

Backup files are stored under the following prefix:

```text
backups/
```

The exported objects are organized using UTC dates and unique filenames.

Example:

```text
backups/
└── 2026-11-05/
    ├── users-12-30-00-123456-REQUEST_ID.json
    ├── users-12-32-00-123456-REQUEST_ID.json
    └── users-12-34-00-123456-REQUEST_ID.json
```

S3 prefixes are created automatically when objects are uploaded. There is no need to manually create the `backups/` folder.

### 3. Import User Data Using AWS Lambda

Created and deployed the first Lambda function:

`import-jsonplaceholder-users`

**Function Responsibilities:**

1. Sends an HTTP GET request to `https://jsonplaceholder.typicode.com/users`.
2. Retrieves 10 sample user records.
3. Processes the API response.
4. Connects to DynamoDB using Boto3.
5. Inserts each user record into the `jsonplaceholder-users` table using `PutItem`.

**Environment Variable:**

```text
TABLE_NAME=jsonplaceholder-users
```

**Required IAM Permission:**

- `dynamodb:PutItem`

**Testing Result:**

The Lambda function executed successfully.

Expected response:

```json
{
  "statusCode": 200,
  "body": "{\"table\": \"jsonplaceholder-users\", \"saved_count\": 10}"
}
```

I also verified the inserted records directly in DynamoDB and inspected the execution logs in CloudWatch.

### 4. Export DynamoDB Records to Amazon S3

Created and deployed the second Lambda function:

`export-dynamodb-to-s3`

This function reads records from DynamoDB and exports them to Amazon S3 as JSON backup files.

**Function Responsibilities:**

1. Connects to the DynamoDB table using Boto3.
2. Scans the table to retrieve user records.
3. Handles DynamoDB Scan pagination to retrieve all available records.
4. Converts DynamoDB Decimal values into JSON-compatible values.
5. Serializes the records into JSON format.
6. Generates a unique backup filename.
7. Uploads the JSON backup file to Amazon S3.

**Environment Variables:**

```text
TABLE_NAME=jsonplaceholder-users
BUCKET_NAME=student-lambda-backups-createdbytomal
BACKUP_PREFIX=backups
```

**Required IAM Permissions:**

- `dynamodb:Scan`
- `s3:PutObject`

**Backup File Naming Convention:**

Each execution generates a unique S3 object key using:

- UTC date
- UTC time, including microseconds
- Lambda invocation request ID

Example:

```text
backups/2026-11-05/users-12-30-00-123456-REQUEST_ID.json
```

This naming convention helps distinguish individual backup executions and prevents different executions from overwriting previous backup files.

### 5. Manually Test the Backup Lambda

Before configuring automatic scheduling, I manually executed the second Lambda function to verify that the DynamoDB-to-S3 export workflow was working correctly.

Expected response:

```json
{
  "statusCode": 200,
  "body": "{\"exported_count\": 10, \"s3_uri\": \"s3://YOUR_BUCKET/backups/...json\", \"scan_pages\": 1}"
}
```

**Verification:**

1. Executed the backup Lambda manually.
2. Checked the Lambda execution result.
3. Opened the S3 bucket.
4. Navigated to the `backups/` prefix.
5. Verified that a new JSON file was uploaded.
6. Downloaded and inspected the JSON file.
7. Executed the Lambda again and verified that another separate backup file was created.

This confirmed that the backup function could successfully retrieve DynamoDB records and store them in S3.

### 6. Configure Amazon EventBridge Scheduler

To automate the backup process, I configured **Amazon EventBridge Scheduler** to invoke the second Lambda function automatically.

**Schedule Configuration:**

| Configuration | Value |
|---|---|
| Schedule Name | `dynamodb-backup-every-2min` |
| Schedule Group | `default` |
| Schedule Type | Recurring |
| Frequency | Rate-based |
| Rate Expression | `rate(2 minutes)` |
| Flexible Time Window | Off |
| Target | `export-dynamodb-to-s3` |
| Initial State | Enabled |

**Scheduler Execution Role:**

A separate IAM execution role was configured for EventBridge Scheduler.

The role grants Scheduler permission to invoke the backup Lambda function through:

```text
lambda:InvokeFunction
```

The Scheduler execution role is different from the execution role of Lambda #2.

**Workflow:**

1. EventBridge Scheduler waits for the configured schedule.
2. Scheduler invokes `export-dynamodb-to-s3`.
3. Lambda #2 scans the DynamoDB table.
4. The retrieved records are converted into JSON.
5. Lambda #2 uploads a new timestamped JSON file to Amazon S3.

The process repeats automatically according to the configured schedule.

**Important:** A two-minute rate represents approximately one invocation every two minutes. It does not guarantee execution at an exact second.

### 7. Verify Automatic S3 Backups

After configuring EventBridge Scheduler, I verified the automated backup workflow.

The verification process involved:

1. Checking that `dynamodb-backup-every-2min` was enabled.
2. Opening Amazon CloudWatch.
3. Inspecting the log group `/aws/lambda/export-dynamodb-to-s3`.
4. Observing new Lambda execution logs generated by scheduled invocations.
5. Opening the S3 bucket.
6. Navigating to `backups/YYYY-MM-DD/`.
7. Verifying that multiple timestamped JSON files were created.

**Expected Backup Frequency:**

If all scheduled executions succeed:

| Time Period | Expected Backup Files |
|---|---:|
| 1 hour | Approximately 30 |
| 1 day | Approximately 720 |
| 30 days | Approximately 21,600 |

The actual number may vary because of delivery delays, retries, execution failures, or schedule configuration.

This demonstrates how EventBridge Scheduler can automate recurring DynamoDB exports without manually executing Lambda #2.

### 8. Monitor Lambda Executions Using CloudWatch

Amazon CloudWatch was used to monitor both Lambda functions.

**CloudWatch Log Groups:**

```text
/aws/lambda/import-jsonplaceholder-users

/aws/lambda/export-dynamodb-to-s3
```

**CloudWatch Metrics:**

The following metrics were inspected:

- `Invocations`
- `Errors`
- `Duration`
- `Throttles` (if available)

CloudWatch Logs provide detailed execution information, while CloudWatch Metrics provide numerical information about Lambda performance and execution.

### 9. Disable the EventBridge Schedule

After completing the automatic backup demonstration and collecting the required evidence, I disabled the EventBridge schedule.

This prevents further scheduled Lambda invocations while preserving the schedule configuration for future use.

**Reasons for disabling the schedule:**

- Prevents unnecessary Lambda executions.
- Avoids continuously generating backup files.
- Helps control AWS usage and potential costs.
- Allows the schedule to be enabled again when required.

The schedule can be re-enabled whenever automated backups are needed.

## Project Structure

```text
DynamoDb_BackUp_in_S3/
│
├── lambda_functions/
│   ├── import_users.py
│   └── backup_to_s3.py
│
├── policies/
│   ├── dynamodbputitempolicy.json
│   └── lambdadynamodbtos3.json
│
├── 01_dynamodb_table_created.png
├── 02_student-lambda-backups-createdbytomal.png
├── 03_lambda_import.png
├── 04_dynamodb_users.png
├── 05_lambda_backup.png
├── 06_s3_backup.png
├── 07_couple_of_s3_backup.png
├── 08_eventBridge_Schedule.png
├── 09_automatic_s3_backups.png
├── 10_evenbridge_disabled.png
│
└── README.md
```

## Screenshots and Implementation Evidence

The following screenshots document the AWS resources, configurations, Lambda executions, and automated backup workflow.

| Screenshot | Description |
|---|---|
| `01_dynamodb_table_created.png` | Creation of the DynamoDB table |
| `02_student-lambda-backups-createdbytomal.png` | Creation of the private S3 bucket |
| `03_lambda_import.png` | Lambda #1 execution and API data import |
| `04_dynamodb_users.png` | Imported user records in DynamoDB |
| `05_lambda_backup.png` | Lambda #2 execution |
| `06_s3_backup.png` | JSON backup file uploaded to S3 |
| `07_couple_of_s3_backup.png` | Multiple timestamped backup files |
| `08_eventBridge_Schedule.png` | EventBridge Scheduler configuration |
| `09_automatic_s3_backups.png` | Automatically generated S3 backup files |
| `10_evenbridge_disabled.png` | Disabled EventBridge schedule |

## IAM and Security

IAM roles and policies were configured to control access between AWS services.

Separate execution roles were used for the two Lambda functions and EventBridge Scheduler.

### Lambda #1 – Import Users

Required permission:

- `dynamodb:PutItem` – Allows inserting user records into the DynamoDB table.

### Lambda #2 – Backup to S3

Required permissions:

- `dynamodb:Scan` – Allows reading records from the DynamoDB table.
- `s3:PutObject` – Allows uploading JSON backup files to the S3 bucket.

### EventBridge Scheduler

Required permission:

- `lambda:InvokeFunction` – Allows EventBridge Scheduler to invoke the backup Lambda function.

### Additional Security Measures

- S3 Block Public Access was enabled.
- The S3 bucket remained private.
- Default S3 server-side encryption was enabled.
- IAM permissions were restricted to the required AWS resources and actions.
- AWS credentials were not hardcoded into Python source code.
- Boto3 used the Lambda execution role credentials to access AWS services.
- Separate execution roles were maintained for different responsibilities.

These configurations follow the principle of least privilege.

## Key Concepts and Technical Questions

This section explains important AWS concepts and technical decisions encountered during the implementation of this project.

### 1. What is the difference between Lambda's execution role and Scheduler's execution role?

Both are IAM roles, but they serve different purposes.

**Lambda Execution Role**

The Lambda execution role is assumed by AWS Lambda when a function executes. It determines which AWS resources the function's code can access.

For example:

- Lambda #1 requires `dynamodb:PutItem` to insert records into DynamoDB.
- Lambda #2 requires `dynamodb:Scan` to retrieve records from DynamoDB.
- Lambda #2 also requires `s3:PutObject` to upload backup files to S3.

**EventBridge Scheduler Execution Role**

The Scheduler execution role is assumed by Amazon EventBridge Scheduler when it triggers a scheduled target.

In this project, it allows EventBridge Scheduler to invoke Lambda #2 using:

- `lambda:InvokeFunction`

**Key Differences:**

| Lambda Execution Role | Scheduler Execution Role |
|---|---|
| Assumed by AWS Lambda | Assumed by EventBridge Scheduler |
| Controls what Lambda code can access | Controls which target Scheduler can invoke |
| Allows DynamoDB and S3 operations | Allows invoking Lambda #2 |
| Used during Lambda execution | Used during scheduled invocation |

In simple terms, Scheduler's role allows the scheduled invocation to happen, while Lambda's role allows the function to perform its actual work.

### 2. Why does running Lambda #1 twice not produce 20 DynamoDB users?

Lambda #1 retrieves 10 user records from JSONPlaceholder and inserts them into DynamoDB.

Running the function twice does not create 20 records because DynamoDB uses a primary key to uniquely identify each item.

In this project, the partition key is:

```text
id (Number)
```

JSONPlaceholder returns the same user IDs when the function is executed again.

When Lambda uses DynamoDB's `PutItem` operation with an existing partition key, DynamoDB replaces the existing item rather than creating another item with the same primary key.

**First Execution:**

```text
User ID: 1
User ID: 2
User ID: 3
...
User ID: 10
```

DynamoDB contains 10 items.

**Second Execution:**

The same 10 user IDs are inserted again.

DynamoDB replaces the existing items with the new items instead of creating duplicates.

**Final Result:**

```text
First execution  = 10 items
Second execution = 10 items
Total            = 10 items
```

This behavior makes the import operation effectively idempotent for this dataset.

An operation is called idempotent when repeating it with the same input produces the same final state rather than repeatedly creating additional records.

### 3. Why do Lambda #2 backups have unique timestamps and request IDs?

Lambda #2 exports DynamoDB records to S3 using unique backup filenames.

Each Lambda invocation has a unique AWS request ID, available through the Lambda execution context.

The backup filename includes:

- UTC date.
- UTC time, including microseconds.
- Lambda invocation request ID.

For example:

```text
backups/
    2026-11-05/
        users-12-30-00-123456-REQUEST_ID.json
        users-12-32-00-123456-REQUEST_ID.json
        users-12-34-00-123456-REQUEST_ID.json
```

**Why use timestamps?**

- Identify when a backup was generated.
- Organize backup files chronologically.
- Easily locate a particular backup.

**Why use request IDs?**

- Uniquely identify individual Lambda invocations.
- Distinguish separate executions.
- Help correlate S3 backup files with CloudWatch logs.
- Avoid accidental filename collisions.

Combining timestamps and request IDs allows each successful backup execution to create a separate S3 object instead of overwriting previous files.

### 4. What is the difference between `print()` logs and `Invocations` / `Errors` metrics?

Amazon CloudWatch provides both logs and metrics for monitoring Lambda functions. However, they serve different purposes.

**A. print() Logs**

Python's `print()` statements generate log messages that are sent to Amazon CloudWatch Logs.

For example:

```python
print("Starting DynamoDB backup...")
print("Exported 10 users to S3")
```

These messages help developers understand what happened during a particular execution.

Logs can provide:

- Execution progress.
- Debugging information.
- Error messages.
- Custom application messages.
- Information about uploaded backup files.

**B. Invocations Metric**

The `Invocations` metric indicates how many times the Lambda function was invoked.

For example:

```text
Invocations = 10
```

This means the function received 10 invocations during the selected monitoring period.

**C. Errors Metric**

The `Errors` metric indicates how many invocations resulted in a function error.

For example:

```text
Invocations = 10
Errors      = 2
```

This means the function was invoked 10 times and 2 invocations resulted in errors.

**Key Differences:**

| Feature | print() Logs | Invocations Metric | Errors Metric |
|---|---|---|---|
| Type | Log messages | Metric | Metric |
| Purpose | Understand execution details | Count executions | Count failed executions |
| Location | CloudWatch Logs | CloudWatch Metrics | CloudWatch Metrics |
| Custom messages | Yes | No | No |
| Debugging | Detailed | Limited | Limited |

In simple terms, logs explain what happened inside the function, while metrics provide numerical information about execution and failures.

A function can have successful invocations without printing any messages. Similarly, printing a success message does not independently prove that the entire backup workflow completed successfully.

### 5. What would happen if you deleted the DynamoDB table but kept its JSON files in S3? Could the original indexes/configuration be automatically restored from the JSON alone?

If the DynamoDB table were deleted but the JSON backup files remained in S3, the exported data would still be available.

However, the original DynamoDB table and its configuration would not automatically be restored from those JSON files.

**What can be recovered?**

The JSON backup files contain the exported user records.

For example:

```json
{
    "id": 1,
    "name": "Leanne Graham",
    "username": "Bret",
    "email": "Sincere@april.biz"
}
```

These records can be used to populate a newly created DynamoDB table.

**What cannot automatically be recovered from JSON data alone?**

- Original partition key and sort key definitions.
- Global Secondary Indexes (GSIs).
- Local Secondary Indexes (LSIs).
- Table capacity and billing configuration.
- TTL configuration.
- DynamoDB Streams settings.
- Point-in-Time Recovery settings.
- Other table-level configurations.

The JSON export contains item data, not the complete DynamoDB table definition.

**How can the table be restored?**

A recovery process would involve:

1. Recreating the DynamoDB table.
2. Configuring the original partition key and sort key.
3. Recreating any required secondary indexes.
4. Restoring other required table configurations.
5. Reading the JSON backup files from S3.
6. Importing the records into the newly created table using Boto3.

For production environments, infrastructure-as-code tools such as Terraform or AWS CloudFormation can help preserve and recreate the original infrastructure configuration.

**Important:** A JSON data export and a complete DynamoDB infrastructure backup are two different things.

### 6. Why is an export every 2 minutes usually a poor production backup strategy for large DynamoDB tables?

Running a full DynamoDB table Scan and exporting all records to S3 every two minutes may work for a small educational project, but it is generally inefficient for large production databases.

There are several reasons.

**A. High Read Consumption**

A full DynamoDB Scan reads records across the table.

Repeating this operation every two minutes can consume significant read capacity and potentially affect application performance.

**B. Increased AWS Costs**

Frequent scans and repeated S3 uploads can increase AWS costs.

As the database grows, repeatedly exporting the entire dataset becomes increasingly expensive.

**C. Lambda Execution Time Limitations**

Large DynamoDB tables may require substantial time to scan and export.

Lambda functions have a maximum execution timeout of 15 minutes. A sufficiently large export may exceed the available execution time.

**D. Overlapping Executions**

If one backup takes longer than two minutes, another scheduled execution may begin before the previous backup finishes.

This can create overlapping executions and additional resource consumption.

**E. Duplicate Data and Storage Growth**

Each full export creates another copy of the entire dataset.

Frequent exports can rapidly increase S3 storage consumption, especially when old backup files are never removed.

**F. Backup Consistency**

A DynamoDB Scan does not automatically provide a point-in-time snapshot of the entire table.

A Scan is eventually consistent by default. Even enabling strongly consistent reads does not guarantee a transactionally consistent snapshot across the entire table.

If records change while the scan is running, the exported data may not represent one consistent point in time.

**Better Production Alternatives**

For larger production workloads, consider:

- **DynamoDB Point-in-Time Recovery (PITR):** Provides continuous backups and supports recovery to a selected point in time within the supported recovery window.
- **DynamoDB Export to S3:** Uses DynamoDB's managed export capability to export table data to S3 without consuming table read capacity. PITR must be enabled.
- **Incremental Exports:** Export changes over a selected time period instead of repeatedly exporting the entire table.
- **S3 Lifecycle Policies:** Automatically transition or expire older backup objects according to retention requirements.

The appropriate backup frequency should depend on the Recovery Point Objective (RPO), Recovery Time Objective (RTO), database size, application requirements, and cost considerations.

**Conclusion:**

A two-minute full-table export is acceptable as a small learning experiment, but production systems should use an appropriate managed backup strategy rather than repeatedly scanning and exporting the entire database.

## Troubleshooting and Common Problems

During AWS implementation, different configuration and permission issues may occur.

The following table summarizes common errors and their possible solutions.

| Problem | Possible Cause | Solution |
|---|---|---|
| `KeyError: TABLE_NAME` | Missing environment variable | Configure `TABLE_NAME` |
| `KeyError: BUCKET_NAME` | Missing S3 bucket variable | Configure `BUCKET_NAME` |
| `AccessDeniedException` on PutItem | Incorrect IAM permissions | Verify `dynamodb:PutItem` |
| `AccessDeniedException` on Scan | Incorrect backup Lambda policy | Verify `dynamodb:Scan` |
| S3 Access Denied | Incorrect bucket or prefix ARN | Verify `s3:PutObject` permissions |
| `ResourceNotFoundException` | Incorrect table name or Region | Verify the table and AWS Region |
| `NoSuchBucket` | Incorrect bucket name | Verify `BUCKET_NAME` |
| DynamoDB ValidationException | Incorrect primary key type | Use Number for the `id` partition key |
| Unable to import module | Incorrect ZIP structure or handler | Verify ZIP contents and handler |
| Lambda timeout | API or network issue | Check internet access and timeout |
| No CloudWatch logs | Missing logging permissions | Verify `AWSLambdaBasicExecutionRole` |
| Scheduler creates no files | Disabled schedule or incorrect target | Inspect Scheduler configuration and Lambda logs |
| Export contains zero users | DynamoDB table is empty | Execute Lambda #1 and inspect table items |
| Repeated import still shows 10 users | Existing IDs are replaced | Expected `PutItem` behavior |
| Empty or partial export | Scan pagination or execution issue | Check scan handling and CloudWatch logs |

## Cost Management and Resource Cleanup

This project creates AWS resources that may incur charges depending on usage and account eligibility.

Potential charges include:

- DynamoDB read and write requests.
- Lambda invocations and execution duration.
- S3 storage and object requests.
- EventBridge Scheduler invocations.
- CloudWatch log ingestion and storage.

A two-minute schedule can generate approximately 21,600 backup objects in 30 days if every execution succeeds.

Therefore, the schedule should be disabled after the demonstration.

### Cleanup Procedure

After collecting screenshots and documenting the project:

1. Disable the `dynamodb-backup-every-2min` schedule.
2. Delete the schedule if it is no longer required.
3. Delete the Lambda functions if the lab is complete.
4. Delete the `jsonplaceholder-users` DynamoDB table if its data is no longer needed.
5. Delete the JSON objects from the S3 bucket.
6. Delete the S3 bucket after removing its objects.
7. Remove unused CloudWatch log groups if desired.
8. Remove lab-only IAM roles and policies if they are no longer used by other resources.

If S3 Versioning is enabled, remove object versions and delete markers as required.

**Security Reminder:**

- Never expose AWS access keys or secret credentials.
- Keep S3 Block Public Access enabled.
- Do not grant unnecessary `s3:*` permissions.
- Follow the principle of least privilege.
- Verify that no unwanted scheduled resources remain after cleanup.

## What I Learned

Through this hands-on project, I gained practical experience with:

- Creating and configuring Amazon DynamoDB tables.
- Understanding NoSQL databases and partition keys.
- Retrieving external API data using Python.
- Deploying Python applications using AWS Lambda.
- Using Boto3 to interact with DynamoDB and S3.
- Understanding DynamoDB `PutItem` and `Scan` operations.
- Handling DynamoDB Scan pagination.
- Creating private S3 buckets and storing JSON objects.
- Generating unique timestamped backup filenames.
- Configuring Lambda environment variables.
- Creating IAM execution roles and attaching permissions.
- Understanding the difference between Lambda and Scheduler execution roles.
- Using Amazon EventBridge Scheduler for recurring Lambda invocations.
- Automating DynamoDB-to-S3 export workflows.
- Monitoring Lambda executions using CloudWatch Logs and Metrics.
- Understanding Lambda request IDs, Invocations, and Errors.
- Troubleshooting IAM permission and DynamoDB validation errors.
- Understanding the limitations of JSON exports compared with production-grade database backups.
- Exploring DynamoDB Point-in-Time Recovery and managed S3 exports.

## Limitations and Future Improvements

This project is an educational demonstration of a scheduled DynamoDB-to-S3 JSON export workflow, not a production-grade backup solution.

Possible improvements include:

- Implementing a restore function to import JSON backup files into DynamoDB.
- Adding automated backup retention using S3 Lifecycle rules.
- Configuring CloudWatch alarms and failure notifications.
- Implementing additional error handling and retry mechanisms.
- Adding backup validation and integrity checks.
- Automating infrastructure provisioning using Terraform or AWS CloudFormation.
- Creating a CI/CD pipeline using GitHub Actions.
- Exploring DynamoDB Point-in-Time Recovery (PITR).
- Implementing DynamoDB managed exports to S3.
- Preserving DynamoDB table definitions and indexes using infrastructure-as-code tools.
- Implementing incremental backup strategies for larger datasets.

## Acknowledgment

This project was completed as part of the **BongoDev DevOps Bootcamp**.

The assignment helped me strengthen my understanding of AWS serverless services through practical implementation, deployment, testing, troubleshooting, automated scheduling, monitoring, and documentation.

