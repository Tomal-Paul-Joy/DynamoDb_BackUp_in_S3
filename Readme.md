
# AWS Serverless DynamoDB Backup to S3

A hands-on AWS project demonstrating a serverless data pipeline using **AWS Lambda, Amazon DynamoDB, Amazon S3, Python, and Boto3**.

This project was completed as part of my **BongoDev DevOps Bootcamp** to gain practical experience with AWS services, serverless computing, IAM permissions, and data backup workflows.

## Project Overview

The objective of this project was to retrieve sample user information from a public REST API, store the data in Amazon DynamoDB, and export the stored records as JSON backup files to Amazon S3.

The workflow uses two AWS Lambda functions:

- **Import Lambda:** Retrieves user information from the JSONPlaceholder API and stores it in DynamoDB.
- **Backup Lambda:** Reads records from DynamoDB and exports them to Amazon S3 as timestamped JSON files.

## Architecture

```text
             JSONPlaceholder API
                     |
                     v
             AWS Lambda (Import)
                     |
                     v
              Amazon DynamoDB
             jsonplaceholder-users
                     |
                     v
             AWS Lambda (Backup)
                     |
                     v
                 Amazon S3
              JSON Backup Files
```

## Technologies Used

| Technology | Purpose |
|---|---|
| AWS Lambda | Execute serverless Python functions |
| Amazon DynamoDB | Store user information |
| Amazon S3 | Store JSON backup files |
| AWS IAM | Manage permissions between AWS services |
| Amazon CloudWatch | Monitor Lambda executions and logs |
| Python | Implement Lambda functions |
| Boto3 | Interact with AWS services using Python |
| JSONPlaceholder API | Provide sample user data |

## AWS Configuration

| Resource | Configuration |
|---|---|
| AWS Region | `us-west-2` (US West – Oregon) |
| DynamoDB Table | `jsonplaceholder-users` |
| Partition Key | `id` (Number) |
| S3 Bucket | `student-lambda-backups-createdbytomal` |
| Import Lambda | `import-jsonplaceholder-users` |
| Backup Lambda | `export-dynamodb-to-s3` |

## Implementation Steps

### 1. Amazon DynamoDB Setup

Created a DynamoDB table named `jsonplaceholder-users`.

**Configuration:**
- Table name: `jsonplaceholder-users`
- Partition key: `id`
- Data type: Number
- No sort key

The table stores user information retrieved from the JSONPlaceholder API.

### 2. Amazon S3 Bucket Setup

Created an Amazon S3 bucket named `student-lambda-backups-createdbytomal` to store exported JSON backup files.

**Configuration:**
- AWS Region: `us-west-2`
- Block Public Access: Enabled
- Bucket access: Private

Backup objects are stored under the following prefix:

```text
backups/
```

### 3. Import User Data Using AWS Lambda

Created and deployed the first Lambda function:

`import-jsonplaceholder-users`

The function performs the following operations:

1. Sends an HTTP request to the JSONPlaceholder API.
2. Retrieves 10 sample user records.
3. Processes the API response.
4. Connects to DynamoDB using Boto3.
5. Inserts the records into the `jsonplaceholder-users` table.

**Environment variable:**

```text
TABLE_NAME=jsonplaceholder-users
```

**IAM permission:**

- `dynamodb:PutItem`

**Testing result:**

The Lambda test completed successfully:

- Status code: `200`
- Saved records: `10`

I also verified the inserted records directly in DynamoDB.

### 4. Export DynamoDB Records to Amazon S3

Created a second Lambda function:

`export-dynamodb-to-s3`

This function reads records from DynamoDB and exports them to Amazon S3 as JSON files.

The main operations are:

1. Connects to the DynamoDB table using Boto3.
2. Scans the table to retrieve user records.
3. Handles the returned DynamoDB data.
4. Converts the records into JSON-compatible data.
5. Uploads the JSON backup file to S3.

**Environment variables:**

```text
TABLE_NAME=jsonplaceholder-users
BUCKET_NAME=student-lambda-backups-createdbytomal
BACKUP_PREFIX=backups
```

**IAM permissions:**

- `dynamodb:Scan`
- `s3:PutObject`

### 5. Verify S3 Backup Files

After deploying the backup Lambda, I executed it manually to verify that DynamoDB records were exported successfully.

I checked the S3 bucket and confirmed that the JSON backup file was uploaded.

I also executed the backup function multiple times and verified that separate timestamped backup files were created instead of overwriting previous backups.

## Project Structure

```text
DynamoDb_BackUp_in_S3/
│
├── lambda_functions/
│   ├── import_users.py
│   └── backup_to_s3.py
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
│
└── README.md
```

## IAM and Security

IAM roles and policies were configured to control access between AWS services.

Separate Lambda execution roles were used for the import and backup functions.

- The import Lambda was granted `dynamodb:PutItem` permission to insert records into the DynamoDB table.
- The backup Lambda was granted `dynamodb:Scan` permission to read records and `s3:PutObject` permission to upload backup files.
- The S3 bucket was configured with public access blocked.

Account-specific resource identifiers were used in IAM policies to restrict access to the required AWS resources.

## What I Learned

Through this hands-on project, I gained practical experience with:

- Creating and configuring Amazon DynamoDB tables.
- Creating private S3 buckets and storing backup objects.
- Deploying Python applications using AWS Lambda.
- Using Boto3 to interact with DynamoDB and S3.
- Configuring Lambda environment variables.
- Creating IAM execution roles and attaching permissions.
- Understanding how AWS services communicate through IAM permissions.
- Testing Lambda functions and verifying stored data.
- Troubleshooting AWS errors such as `AccessDeniedException` and DynamoDB validation errors.
- Understanding the importance of verifying actual backup files rather than relying only on successful Lambda responses.

## Limitations and Future Improvements

This project is an educational demonstration of a basic DynamoDB-to-S3 backup workflow, not a production-grade backup solution.

Possible improvements include:

- Integrating Amazon EventBridge Scheduler for recurring backups.
- Adding automated backup retention using S3 Lifecycle rules.
- Configuring CloudWatch alarms and failure notifications.
- Implementing a restore function to import JSON backups into DynamoDB.
- Automating infrastructure provisioning using Terraform.
- Creating a CI/CD pipeline using GitHub Actions.
- Exploring DynamoDB Point-in-Time Recovery (PITR) and managed exports to S3 for production backup requirements.

## Acknowledgment

This project was completed as part of the **BongoDev DevOps Bootcamp**.

The project helped me strengthen my understanding of AWS serverless services through practical implementation, deployment, testing, troubleshooting, and documentation.
