---
title : "Automated Reporting & Data Backup"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

In this section, we will use **Amazon EventBridge Scheduler** to automatically send financial reports and back up **SecureTable** data.

SecureTable uses two automated workflows:

- **Reporting:** EventBridge Scheduler → `SecureTable-Report` → DynamoDB → Amazon SNS → Admin Email.
- **Backup:** EventBridge Scheduler → `SecureTable-Backup` → DynamoDB Export to S3 → Amazon S3 → S3 Glacier Flexible Retrieval.

All resources are configured in **Asia Pacific (Singapore) – `ap-southeast-1`**. The schedules use the **`Asia/Ho_Chi_Minh`** time zone.

## 1. Create an Amazon SNS Topic for Report Delivery

We use Amazon SNS to send reports from Lambda to the administrator's email address.

1. Open **Amazon SNS → Topics → Create topic**.
2. Select **Type: Standard**.
3. Enter **Name:** `SecureTable-FinancialReports`.
4. Keep the settings appropriate for the demo environment, then select **Create topic**.
5. Copy the **Topic ARN**. It has the following format:

   ```text
   arn:aws:sns:ap-southeast-1:<YOUR_ACCOUNT_ID>:SecureTable-FinancialReports
   ```

![Create SNS Topic](../../images/5-Workshop/5.7-Automation/sns-topic.png)

6. Open the newly created topic → **Create subscription**.
7. Select **Protocol: Email**, enter the email address that will receive reports in the **Endpoint** field, then select **Create subscription**.
8. Check your inbox, open the confirmation email from AWS, and select **Confirm subscription**.
9. Return to SNS and verify that the subscription status is **Confirmed**.

![SNS Email Subscription](../../images/5-Workshop/5.7-Automation/sns-subscription.png)

**Note:** If the subscription remains in the `Pending confirmation` state, SNS will not send reports to that email address. You can use **Publish message** in the SNS Topic to verify email delivery before connecting Lambda.

## 2. Configure and Test the Report Lambda

The `SecureTable-Report` Lambda reads data from the business-related DynamoDB tables, generates reports, and sends their contents through SNS. The `SecureTable-ReportDispatch` table tracks report delivery status and helps prevent duplicate reports.

### 2.1. Configure the Report Lambda

1. Open **AWS Lambda → Functions → SecureTable-Report**.
2. Select **Configuration → Environment variables → Edit**.
3. Add the following environment variable:

   | Key | Value |
   |---|---|
   | `SNS_TOPIC_ARN` | The ARN of the `SecureTable-FinancialReports` topic created in Step 1 |

4. Select **Save**.

![Report Lambda Configuration](../../images/5-Workshop/5.7-Automation/report-config.png)

### 2.2. Test Report Delivery

1. In `SecureTable-Report`, open **Test → Create new event**.
2. Create a test event with `dry_run` enabled:

   ```json
   {
     "report_type": "DAILY",
     "business_date": "2026-11-27",
     "dry_run": true
   }
   ```

3. Select **Test**, then review the response and **CloudWatch Logs** to check the generated report data. Change `business_date` to an actual date for which the system has data.
4. After confirming that the results are valid and the email subscription has been confirmed, you can test actual report delivery by changing `"dry_run": false`.
5. Check the received email and the corresponding record in `SecureTable-ReportDispatch`.

![Report Lambda Test](../../images/5-Workshop/5.7-Automation/report-test.png)

**Note:** `dry_run: true` only tests report generation; it does not mean the report was successfully delivered. When performing an actual send, check the SNS response and the status in `SecureTable-ReportDispatch`. Avoid repeatedly sending reports for the same reporting period; the deduplication mechanism is designed to help prevent duplicates.

## 3. Create EventBridge Schedules for Daily and Weekly Reports

We will create **two schedules** that both invoke `SecureTable-Report`: one daily and one weekly.

1. Open **Amazon EventBridge → Scheduler → Create schedule**.
2. Name the schedule **`SecureTable-Daily-EmailReport`**.
3. Select **Recurring schedule** and configure the following settings:

   | Setting | Value |
   |---|---|
   | **Schedule type** | Cron-based schedule |
   | **Cron expression** | `cron(0 22 * * ? *)` |
   | **Time zone** | `Asia/Ho_Chi_Minh` |
   | **Flexible time window** | Off |
   | **Schedule state** | Enabled |

4. Under **Target**, select **AWS Lambda → Invoke**, then choose `SecureTable-Report`.
5. Configure the **Input / Payload**:

   ```json
   {
     "report_type": "DAILY",
     "scheduled_at": "<aws.scheduler.scheduled-time>"
   }
   ```

6. Under **Execution role**, select **Create new role for this schedule** so that EventBridge Scheduler can invoke `SecureTable-Report` using the `lambda:InvokeFunction` permission.
7. Complete the process by selecting **Create schedule**.

![Daily Report Scheduler](../../images/5-Workshop/5.7-Automation/scheduler-daily-report.png)

8. Create the second schedule with these settings:

   | Setting | Value |
   |---|---|
   | **Name** | `SecureTable-Weekly-EmailReport` |
   | **Cron expression** | `cron(15 22 ? * SUN *)` |
   | **Time zone** | `Asia/Ho_Chi_Minh` |
   | **Flexible time window** | Off |
   | **Target** | `SecureTable-Report` |

9. Enter the payload for the weekly report:

   ```json
   {
     "report_type": "WEEKLY",
     "scheduled_at": "<aws.scheduler.scheduled-time>"
   }
   ```

10. Create an Execution Role for this schedule when prompted by the Console, then complete the setup.

**Note:** The string `<aws.scheduler.scheduled-time>` is a **context attribute** that EventBridge Scheduler replaces with the scheduled execution time. Although the schedule uses the Vietnam time zone, the timestamp passed in the payload may be in UTC; Lambda must convert it to the correct reporting date. **The Scheduler Execution Role is different from the Lambda Execution Role**: Scheduler needs permission to invoke Lambda, while Lambda needs permissions to read DynamoDB and publish messages to SNS.

## 4. Enable Point-in-Time Recovery (PITR) for DynamoDB

To export DynamoDB data to S3, we must enable **Point-in-Time Recovery (PITR)** on the tables included in the backup scope.

1. Open **Amazon DynamoDB → Tables**.
2. Select a table, for example `SecureTable-Bills`.
3. Open **Backups → Point-in-time recovery (PITR)** and select **Turn on**.
4. Save the configuration and verify that its status changes to **Enabled**.
5. Repeat these steps for the following **13 tables**:

| No. | DynamoDB Table |
|---:|---|
| 1 | `SecureTable-Bills` |
| 2 | `SecureTable-Orders` |
| 3 | `SecureTable-OrderItems` |
| 4 | `SecureTable-Ingredients` |
| 5 | `SecureTable-InventoryReceipts` |
| 6 | `SecureTable-InventoryCorrections` |
| 7 | `SecureTable-InventoryMovements` |
| 8 | `SecureTable-FinanceLedger` |
| 9 | `SecureTable-InventoryCounts` |
| 10 | `SecureTable-AuditLogs` |
| 11 | `SecureTable-Tables` |
| 12 | `SecureTable-MenuItems` |
| 13 | `SecureTable-ReceiptCorrectionState` |

**Note:** Not all **18 tables** are exported by this backup schedule. Supporting tables such as `GuestSessions`, `Idempotency`, `PendingByIngredient`, `ReportDispatch`, and `NotificationOutbox` are excluded from the list of 13 tables. PITR incurs charges while enabled, **even if no export has been performed**. After enabling PITR for the first time, you may need to wait until the recovery window covers the point in time requested by the Backup Lambda.

## 5. Create an S3 Bucket and Configure a Lifecycle Rule for Glacier

The Backup Lambda will instruct DynamoDB to export data to a **separate S3 bucket**, rather than the bucket hosting the frontend.

### 5.1. Create an S3 Bucket for Backups

1. Open **Amazon S3 → General purpose buckets → Create bucket**.
2. Select the **Asia Pacific (Singapore) – `ap-southeast-1`** Region.
3. Enter a unique bucket name, such as `<YOUR_BACKUP_BUCKET>` (replace it with a valid name that is not already in use).
4. Keep **Block all public access** enabled.
5. Use the default **SSE-S3** encryption, then select **Create bucket**.

### 5.2. Create an S3 Lifecycle Rule

1. Open the newly created bucket → **Management → Create lifecycle rule**.
2. Name the rule, for example `SecureTable-ArchiveToGlacier`.
3. Limit its scope using **Prefix:** `securetable/weekly/`.
4. Select **Transition current versions of objects between storage classes**.
5. Configure the following settings:

   | Setting | Value |
   |---|---|
   | **Choose storage class transitions** | `S3 Glacier Flexible Retrieval` |
   | **Transition after** | 30 days |
   | **Expire after** | 180 days |

![S3 Lifecycle Rule](../../images/5-Workshop/5.7-Automation/s3-lifecycle.png)

**Note:** By default, new S3 Lifecycle Rules **do not transition objects smaller than 128 KB** to another storage class. Some manifest files generated by DynamoDB exports may fall below this threshold. **S3 Glacier Flexible Retrieval** also has transition fees, retrieval charges, and a minimum billable storage duration.

## 6. Configure and Test the Backup Lambda

1. Open **AWS Lambda → Functions → SecureTable-Backup → Configuration → Environment variables → Edit**.
2. Add or update the following variables:

   | Key | Value |
   |---|---|
   | `BACKUP_BUCKET` | The actual S3 backup bucket name, without `s3://` |
   | `BACKUP_PREFIX` | `securetable/weekly` |

3. Open **Configuration → General configuration** and use **Memory: 256 MB** and **Timeout: 120 seconds** as initial settings.
4. Open **Configuration → Permissions** and verify that the Inline Policy for `SecureTable-Backup` was obtained from `Lambda/Backup/SecureTable-Backup.json`.
5. Verify that `<YOUR_ACCOUNT_ID>` and `<YOUR_BACKUP_BUCKET>` have been replaced in the JSON, and that the permissions cover exports for the correct 13 tables and writing objects under `securetable/weekly/*`.

![Backup Lambda Configuration](../../images/5-Workshop/5.7-Automation/backup-config.png)

### 6.1. Test Without Creating an Export

1. Open **Test → Create new event** for the Backup Lambda.
2. Enter the following payload:

   ```json
   {
     "dry_run": true,
     "backup_id": "demo-2026-11-28"
   }
   ```

3. Select **Test**.
4. Verify that the result includes `ready: true` and `table_count: 13`. If `ready: false`, check the list of missing tables or tables that do not have PITR enabled.

### 6.2. Test an Actual Export

Once all 13 tables are ready, you can test an actual export with the following payload:

```json
{
  "dry_run": false,
  "backup_id": "demo-2026-11-28"
}
```

The Backup Lambda will request a **DynamoDB Export to S3** for each table. The expected initiation response includes `mode: STARTED`, `exports_requested: 13`, and the `export_arn` values.

1. Open **DynamoDB → Exports to S3** to check the status of the export jobs.
2. Wait for each export to reach **COMPLETED**; if any export shows **FAILED**, inspect its error details and CloudWatch Logs.
3. Open **S3 → backup bucket** and find the data under the corresponding prefix:

   ```text
   securetable/weekly/demo-2026-11-28/<table-name>/AWSDynamoDB/...
   ```

4. Open the export folders to inspect the manifest files and exported data.

![DynamoDB Export Status](../../images/5-Workshop/5.7-Automation/dynamodb-export.png)
![Exported Data in S3](../../images/5-Workshop/5.7-Automation/s3-export-objects.png)

## 7. Create an EventBridge Scheduler for Weekly Backups

1. Open **Amazon EventBridge → Scheduler → Create schedule**.
2. Enter **Name:** `SecureTable-Weekly-DynamoBackup`.
3. Select **Recurring schedule** and configure the following settings:

   | Setting | Value |
   |---|---|
   | **Cron expression** | `cron(0 2 ? * SUN *)` |
   | **Time zone** | `Asia/Ho_Chi_Minh` |
   | **Flexible time window** | Off |
   | **Target** | AWS Lambda → `SecureTable-Backup` |
   | **Schedule state** | Enabled |

4. Under **Input / Payload**, enter:

   ```json
   {
     "scheduled_at": "<aws.scheduler.scheduled-time>"
   }
   ```

5. Under **Execution role**, select **Create new role for this schedule**, allowing EventBridge Scheduler to invoke only the `SecureTable-Backup` Lambda.
6. Review the settings and select **Create schedule**.

![Weekly Backup Scheduler](../../images/5-Workshop/5.7-Automation/scheduler-backup.png)

With this schedule, the Backup Lambda runs at **02:00 on Sundays (Vietnam time)**. Lambda selects a point in time within the PITR recovery window before initiating the exports. The DynamoDB Export jobs continue running after Lambda finishes.

After completing these steps, **SecureTable automatically delivers reports through SNS and periodically backs up 13 DynamoDB tables to Amazon S3**. In the next section, **5.8 – Security Hardening & System Monitoring**, we will review CloudWatch, error alarms, and additional security configurations.
