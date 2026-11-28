---
title : "Security Hardening & System Monitoring"
date : 2024-01-01
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---

SecureTable uses **CloudFront + S3**, **Cognito + IAM**, **API Gateway + Lambda**, and **DynamoDB**. We will use **AWS WAF**, **Amazon CloudWatch**, **Amazon SNS**, and **AWS CloudTrail** to improve observability and issue detection.

## 1. Review Frontend and API Security

### 1.1. Check S3 and CloudFront

1. Open **Amazon S3 → Buckets** and select the bucket hosting the SecureTable frontend.
2. Under **Permissions**, confirm that **Block all public access** is enabled.
3. Verify that the **Bucket policy** grants only the read permissions required by CloudFront through **Origin Access Control (OAC)**; do not enable public read access with `Principal: "*"`.
4. Open **CloudFront → Distributions → select the SecureTable distribution → Origins**.
5. Confirm that the S3 Origin uses OAC and that the API Gateway Origin points to the REST API's `execute-api` domain.
6. Under **Behaviors**, check the following settings:

   | Behavior | Origin | Settings to verify |
   |---|---|---|
   | `Default (*)` | S3 Frontend | `Redirect HTTP to HTTPS`, OAC on the S3 Origin |
   | `/v1/*` | API Gateway | `Redirect HTTP to HTTPS`, `CachingDisabled`, `AllViewerExceptHostHeader` |

**Note:** APIs handling Guest Sessions, orders, and staff data **must not be cached by CloudFront**. CloudFront delivers the frontend over HTTPS, but it does not replace authentication and authorization checks in the backend.

### 1.2. Check Authentication and Authorization

1. Open **Amazon Cognito → User pools → SecureTable-Staff**.
2. Verify that **Self-registration** is disabled if staff accounts are created only by administrators.
3. Confirm the Cognito Groups used by the application: `RECEPTION`, `KITCHEN`, `INVENTORY`, and `ADMIN`.
4. Open **API Gateway → SecureTable-API → Resources** and confirm that Staff Methods use `SecureTableStaffAuthorizer`.
5. For Staff APIs using **Cognito ID Tokens**, leave **Authorization Scopes** empty.
6. Verify that each Lambda function also checks `cognito:groups` before processing business operations.

## 2. Review IAM Permissions Following the Principle of Least Privilege

Each SecureTable Lambda function has its own **Execution Role** and a function-specific **Inline Policy**.

1. Open **AWS Lambda → Functions** and select a function, such as `SecureTable-Ordering`.
2. Select **Configuration → Permissions**, then click the **Execution Role** name to open IAM.
3. Confirm that the role has `AWSLambdaBasicExecutionRole` for CloudWatch Logs and the correct function-specific Inline Policy.
4. Compare the DynamoDB permissions against the tables that the Lambda function actually needs to access.
5. Repeat these checks for important functions such as `SecureTable-Kitchen`, `SecureTable-Report`, and `SecureTable-Backup`.

| Lambda | Permissions to review |
|---|---|
| `SecureTable-Ordering` | Read the menu, bills, and sessions; write orders and reserve ingredients |
| `SecureTable-Kitchen` | Read the queue; update item statuses, inventory, and bills |
| `SecureTable-AdminDashboard` | Read Finance data and Audit Logs |
| `SecureTable-Report` | Read reporting tables, update `ReportDispatch`, and call `sns:Publish` |
| `SecureTable-Backup` | Export 13 DynamoDB tables and write objects to the backup bucket |

**Special check for the Backup Lambda:** The S3 path in the IAM Policy must match the **actual bucket** and the `BACKUP_PREFIX` value configured in Section 5.7. For example, if the bucket is `securetable-backup` and the prefix is `securetable/weekly`, the object-write permission must cover:

```text
arn:aws:s3:::securetable-backup/securetable/weekly/*
```

The corresponding Inline Policy may need permissions including `s3:PutObject`, `s3:PutObjectAcl`, and `s3:AbortMultipartUpload`. Without `s3:PutObject`, Lambda may **successfully initiate export requests**, while the DynamoDB Export jobs still transition to `FAILED` due to `AccessDenied`.

## 3. Check AWS WAF on CloudFront

1. Open **CloudFront → Distributions → select the SecureTable distribution**.
2. Check **Security / AWS WAF** to identify the associated Web ACL (protection pack).
3. Follow the link to **AWS WAF** and select the **CloudFront distributions (Global)** scope.
4. Open the corresponding Web ACL and review **Traffic overview**, active rules, and the number of **Allowed**, **Blocked**, or **Counted** requests.
5. If you add AWS Managed Rules or a Rate-based Rule, consider **monitoring in Count mode first** before switching to Block, to avoid accidentally blocking legitimate ordering or sign-in requests.

**Note:** For a Web ACL protecting CloudFront, AWS WAF uses the **Global / CloudFront** scope (the API for managing these resources uses `us-east-1`), even though SecureTable's Lambda functions and DynamoDB tables are hosted in Singapore. Check feature limits and costs under the current **CloudFront Plan** before enabling additional paid rules.

**Protection limitation:** A WAF associated with CloudFront inspects only requests **that pass through CloudFront**. If API Gateway's default `execute-api` endpoint remains directly accessible, requests sent directly to that endpoint bypass the CloudFront Web ACL. Therefore, keep the **Cognito Authorizer**, Guest Token validation, and Lambda role checks in place.

## 4. Monitor AWS Lambda with CloudWatch

### 4.1. Check Log Groups

AWS Lambda writes logs to CloudWatch when its Execution Role has the appropriate permissions. We will check logs for all 8 Lambda functions:

1. Open **Amazon CloudWatch → Logs → Log Management** in Region `ap-southeast-1`.
2. Find log groups named `/aws/lambda/SecureTable-...`.
3. Open `/aws/lambda/SecureTable-Ordering` or `/aws/lambda/SecureTable-Kitchen`.
4. Select the most recent **Log stream** to inspect execution times, errors, and request-processing details.
5. For each log group, select **Actions → Edit retention setting**, set a retention period such as **14 days** for the demo environment, and save.

![CloudWatch Lambda Logs](../../images/5-Workshop/5.8-Security/cloudwatch-lambda-logs.png)

### 4.2. Check Metrics

1. In AWS Lambda, open `SecureTable-Ordering` → **Monitor → Metrics**.
2. Review the following metrics: **Invocations**, **Errors**, **Duration**, and **Throttles**.
3. Repeat for `SecureTable-Kitchen`, `SecureTable-Inventory`, `SecureTable-Report`, and `SecureTable-Backup`.

| Metric | Meaning |
|---|---|
| `Invocations` | Number of times the Lambda function is invoked |
| `Errors` | Number of executions that fail at the Lambda/runtime level |
| `Duration` | Function execution time |
| `Throttles` | Number of invocation requests rejected due to concurrency limits |

![Lambda Monitoring Metrics](../../images/5-Workshop/5.8-Security/lambda-metrics.png)

**Note:** If a Lambda function handles an error internally and returns JSON with HTTP `400` or `500` without an execution failure, the Lambda **`Errors` metric may not increase**. Therefore, combine Lambda metrics with API Gateway error metrics and application logs rather than relying on a single metric.

## 5. Enable Access Logging for the API Gateway REST API

Access Logs help monitor requests to `SecureTable-API`, including requests rejected at the API Gateway layer.

### 5.1. Prepare CloudWatch Logs Permissions

1. Open **IAM → Roles** and check whether a role for API Gateway to write CloudWatch Logs already exists.
2. If none exists, create an IAM Role with **Trusted entity: API Gateway** (`apigateway.amazonaws.com`) and attach the AWS Managed Policy **`AmazonAPIGatewayPushToCloudWatchLogs`**.
3. Copy the role's ARN.
4. Open **API Gateway → Settings / Account settings** in the Singapore Region and set **CloudWatch log role ARN** to the ARN of that role.

**Note:** This is a **role for API Gateway logging**, separate from the Lambda Execution Roles and EventBridge Scheduler Execution Roles. Do not use a Lambda role for this step.

### 5.2. Configure Access Logging for the `dev` Stage

1. Open **API Gateway → APIs → SecureTable-API → Stages → dev**.
2. Open **Logs and tracing / Logging** and enable **Access logging**.
3. Create or select a CloudWatch Log Group, for example `/aws/apigateway/SecureTable-API-dev-access`.
4. Enter the following JSON format for Access Logs:

   ```json
   {
     "requestId": "$context.requestId",
     "requestTime": "$context.requestTime",
     "httpMethod": "$context.httpMethod",
     "resourcePath": "$context.resourcePath",
     "status": "$context.status",
     "responseLatency": "$context.responseLatency"
   }
   ```

5. Save the configuration. In CloudWatch, set **Log retention: 14 days** for this log group if appropriate.
6. Call a valid API endpoint through CloudFront, then open the log group to confirm that new requests are being recorded.

![API Gateway Access Logging](../../images/5-Workshop/5.8-Security/api-access-logging.png)

**Note:** Log only the fields needed for monitoring. **Do not include the `Authorization` header or request bodies containing session information** in Access Logs. If you also enable Execution Logging, avoid enabling Data tracing in an environment with real data, as it may record sensitive request/response content and increase costs.

## 6. Create CloudWatch Alarms and SNS Notifications

We will create an alarm for Lambda execution errors, using `SecureTable-Ordering` as an example. Other functions can be configured in the same way.

### 6.1. Create an SNS Topic for System Alerts

1. Open **Amazon SNS → Topics → Create topic**.
2. Select **Standard** and enter the name `SecureTable-SystemAlerts`.
3. After creating the topic, select **Create subscription** with **Protocol: Email**.
4. Enter the alert recipient's email address and **Confirm subscription** from the inbox.

### 6.2. Create an Alarm for the Ordering Lambda

1. Open **CloudWatch → Alarms → All alarms → Create alarm**.
2. Select **Select metric → Lambda → By Function Name**.
3. Select `SecureTable-Ordering` → the **Errors** metric.
4. Configure the following settings:

   | Setting | Suggested value |
   |---|---|
   | **Statistic** | `Sum` |
   | **Period** | `5 minutes` |
   | **Threshold type** | `Static` |
   | **Condition** | `Errors >= 1` |
   | **Evaluation periods** | `1` |
   | **Missing data treatment** | `Treat missing data as not breaching` |
   | **Alarm action** | Send to `SecureTable-SystemAlerts` |
   | **Alarm name** | `SecureTable-Ordering-Errors` |

5. Select **Create alarm** and confirm that it appears in the alarm list.
6. You can also create a similar alarm named `SecureTable-Kitchen-Errors` for the Kitchen Lambda.

**Note:** A threshold of `Errors >= 1` over 5 minutes may generate too many alerts as the system grows. CloudWatch sends an SNS notification when the alarm **changes to the relevant state**, not one email for every failed request. Check both **Alarm history** and the confirmed email subscription to verify that alerts are working.

### 6.3. Monitor API Gateway Errors and Automated Tasks

- **API Gateway:** Open **CloudWatch → Metrics → API Gateway** or the Metrics page for `SecureTable-API/dev`. Review `4XXError`, `5XXError`, `Count`, and `Latency`. You can create an alarm for `5XXError` if needed. `4XXError` can also include legitimately rejected requests and does not necessarily indicate a system failure.
- **Report Lambda:** Check CloudWatch Logs, SNS Subscription status, and records in `SecureTable-ReportDispatch` to determine whether reports have been delivered.
- **Backup Lambda:** After running the function, always check **DynamoDB → Exports to S3** until every job reaches `COMPLETED`. **A successful Lambda execution does not mean an export has finished**; S3 write errors such as `AccessDenied` may only appear during the export job.

![Alarms](../../images/5-Workshop/5.8-Security/alarms.png)

## 7. Review AWS CloudTrail and Audit Logs

**AWS CloudTrail** tracks management activity in AWS, while **SecureTable Audit Logs** record business-related changes in the application. These serve different purposes.

1. Open **AWS CloudTrail → Event history** in Region `ap-southeast-1`.
2. Filter by **Event name** or **Event source** to find recent management actions involving Lambda, IAM, DynamoDB, or S3.
3. Open an event to view its **Event time**, **Event name**, **User name**, and related details.
4. Return to SecureTable, sign in as Admin, and open the **Audit Dashboard**.
5. Compare business operations performed in the application, such as stock receipts, adjustments, or bill processing (if the application code records audit entries for those operations).

![AWS CloudTrail Event History](../../images/5-Workshop/5.8-Security/cloudtrail-event-history.png)
![SecureTable Audit Dashboard](../../images/5-Workshop/5.8-Security/audit-dashboard.png)

**Note:** CloudTrail **Event history** shows **management events from the last 90 days** and does not replace business Audit Logs. Event history **does not display data events** such as every read or write of a DynamoDB item. To retain longer histories or record data events, configure an appropriate Trail or CloudTrail Lake setup and consider the associated costs.

## 8. Verify the Results

After completing the steps, verify the following components:

| Component | Expected result |
|---|---|
| S3 + CloudFront | Bucket is not public; OAC and HTTPS work; API behavior does not cache responses |
| Cognito + API Gateway | Staff APIs require valid tokens; Lambda checks groups |
| IAM | All 8 Lambda functions have least-privilege, function-specific permissions; Backup has the correct S3 write permissions |
| AWS WAF | The Web ACL is associated with the correct CloudFront distribution and has observable metrics |
| CloudWatch Logs | New logs appear for Lambda functions and API Gateway Access Logs |
| CloudWatch Alarm | `SecureTable-Ordering-Errors` has been created and is linked to a confirmed SNS Topic |
| CloudTrail | Management events related to AWS resources can be viewed |
| Backup | Export jobs reach `COMPLETED`, rather than stopping at Lambda's `STARTED` response |

Once these steps are complete, SecureTable can **review and restrict access permissions**, **observe logs and metrics**, **receive execution-error alerts**, and **compare AWS management activity with business-level audit history**.
