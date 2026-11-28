---
title : "Resource Cleanup"
date : 2024-01-01
weight : 10
chapter : false
pre : " <b> 5.10. </b> "
---

After completing and testing SecureTable, we will delete AWS resources that are no longer needed to avoid unnecessary charges. All steps will be performed in the **AWS Management Console**. Most resources are located in **Asia Pacific (Singapore) – `ap-southeast-1`**; **CloudFront** and **AWS WAF for CloudFront** are managed globally.

**Note:** The following actions may result in permanent data loss.

## 1. Stop Automated Schedules

To prevent the Report Lambda from continuing to send emails and the Backup Lambda from starting additional export jobs, we will stop EventBridge Scheduler first.

1. Open **Amazon EventBridge → Scheduler → Schedules** in Region `ap-southeast-1`.
2. Find the following schedules:

   | Schedule | Purpose |
   |---|---|
   | `SecureTable-Daily-EmailReport` | Send daily financial reports |
   | `SecureTable-Weekly-EmailReport` | Send weekly financial reports |
   | `SecureTable-Weekly-DynamoBackup` | Back up DynamoDB weekly |

3. Select each schedule and choose **Disable** if you want to pause it temporarily for verification. Then choose **Delete** when you are ready to clean up.
4. Check the schedule list again to confirm that no SecureTable schedules remain.

![Delete EventBridge Schedules](../../images/5-Workshop/5.10-Cleanup/schedules.png)

**Note:** Deleting a schedule does not automatically delete its Lambda function or the **IAM Execution Role** previously used by the Scheduler. We will review those roles in Section 7. If a DynamoDB export job has already been started, deleting the schedule will not automatically cancel that export.

## 2. Delete the CloudFront Distribution and AWS WAF

1. Open **Amazon CloudFront → Distributions** and select the distribution serving the SecureTable website.
2. If the distribution is using the **CloudFront Free Plan or another Pricing Plan**, select **Cancel pricing plan** first. For the Free Plan, cancellation usually takes effect immediately; for paid plans, you may need to wait until the end of the billing cycle.
3. Choose **Disable** for the distribution and wait for the change to finish deploying.
4. Once the distribution is disabled and is no longer associated with a pricing plan, choose **Delete**.
5. Open **AWS WAF → Web ACLs**, select the **CloudFront distributions (Global)** scope, and check the SecureTable Web ACL. If the Web ACL is no longer used, delete it if it was not automatically removed along with the distribution.

![Delete CloudFront Distribution](../../images/5-Workshop/5.10-Cleanup/cloudfront.png)

## 3. Delete API Gateway, Lambda, and Cognito

### 3.1. Delete the REST API

1. Open **Amazon API Gateway → APIs** in `ap-southeast-1`.
2. Select **`SecureTable-API`** → **Delete API**, and confirm.
3. Verify that the API and its `dev` stage no longer appear in the list.

![Delete API Gateway](../../images/5-Workshop/5.10-Cleanup/api.png)

### 3.2. Delete the Eight Lambda Functions

1. Open **AWS Lambda → Functions**.
2. Select and delete each SecureTable function:

   | Lambda Function | Lambda Function |
   |---|---|
   | `SecureTable-GuestServices` | `SecureTable-Ordering` |
   | `SecureTable-Reception` | `SecureTable-Kitchen` |
   | `SecureTable-Inventory` | `SecureTable-AdminDashboard` |
   | `SecureTable-Report` | `SecureTable-Backup` |

3. Confirm that the functions no longer appear in the list.

![Delete Lambda Functions](../../images/5-Workshop/5.10-Cleanup/lambda.png)

### 3.3. Delete the Cognito User Pool

1. Open **Amazon Cognito → User pools**.
2. Select **`SecureTable-Staff`** and choose **Delete user pool**.
3. If **Deletion protection** is enabled, follow the Console instructions to disable it before confirming deletion.

![Delete Cognito](../../images/5-Workshop/5.10-Cleanup/cognito.png)

## 4. Delete the Amazon DynamoDB Tables

1. Open **Amazon DynamoDB → Tables** in `ap-southeast-1`.
2. Filter the list by `SecureTable-` and review the tables associated with the system.
3. Select each table → **Delete**, and confirm as prompted by the Console.
4. Continue until no `SecureTable-*` tables belonging to the project remain.

The tables created in this workshop include:

```text
SecureTable-Tables                  SecureTable-Bills
SecureTable-Orders                  SecureTable-OrderItems
SecureTable-MenuItems               SecureTable-Ingredients
SecureTable-InventoryReceipts       SecureTable-InventoryCorrections
SecureTable-ReceiptCorrectionState  SecureTable-InventoryMovements
SecureTable-InventoryCounts         SecureTable-FinanceLedger
SecureTable-AuditLogs               SecureTable-GuestSessions
SecureTable-Idempotency             SecureTable-PendingByIngredient
SecureTable-NotificationOutbox      SecureTable-ReportDispatch
```

![Delete DynamoDB Tables](../../images/5-Workshop/5.10-Cleanup/dynamodb.png)

**Note:** Deleting a table also deletes its active data. If you still need the data for demonstrations or verification, keep an appropriate backup before deleting it. Exports already stored in S3 are **not automatically deleted** when a DynamoDB table is deleted.

## 5. Delete the SecureTable S3 Buckets

After CloudFront has been deleted and no export jobs are still writing to S3:

1. Open **Amazon S3 → General purpose buckets**.
2. Select the **frontend bucket** created in Section 5.3 → **Empty** to remove the website files.
3. Once the bucket is empty, choose **Delete bucket** and confirm.
4. Repeat the process for the **backup bucket** created in Section 5.7 (for example, `securetable-backup`) if you no longer need the backups.
5. Review the bucket list to confirm that both project buckets have been removed.

![Delete SecureTable S3 Buckets](../../images/5-Workshop/5.10-Cleanup/s3.png)

**Note:** If **S3 Versioning** is enabled, you must also delete all object versions and delete markers. Objects transitioned to **S3 Glacier Flexible Retrieval** have a 90-day minimum billable storage duration; deleting them early may still incur storage charges for the remaining period. Only delete backups when you are certain that you will not need to restore the data.

## 6. Clean Up CloudWatch and SNS

1. Open **Amazon CloudWatch → Alarms → All alarms** and delete the alarms created for SecureTable, such as `SecureTable-Ordering-Errors` and `SecureTable-Kitchen-Errors`.
2. Open **CloudWatch → Logs → Log groups** and delete any log groups you no longer need, including `/aws/lambda/SecureTable-*` and `/aws/apigateway/SecureTable-API-dev-access`.
3. Open **Amazon SNS → Topics** and delete the topics used only by this system:
   - `SecureTable-FinancialReports`
   - `SecureTable-SystemAlerts`
4. Verify that the related subscriptions are no longer needed.

![Delete CloudWatch Logs](../../images/5-Workshop/5.10-Cleanup/logs.png)
![Delete CloudWatch Alarms](../../images/5-Workshop/5.10-Cleanup/alarms.png)
![Delete SNS](../../images/5-Workshop/5.10-Cleanup/sns.png)

**Note:** CloudWatch Logs may continue retaining data and incurring charges after a Lambda function has been deleted. **CloudTrail Event history** does not need to be deleted by default. If you created a dedicated CloudTrail Trail for SecureTable, delete it only after confirming that it is not shared with other resources; any log files stored in S3 and associated Log Groups must still be cleaned up separately.

## 7. Review IAM and Remaining Costs

1. Open **IAM → Roles** and look for Execution Roles used exclusively by the eight SecureTable Lambda functions, the three EventBridge Schedules, and API Gateway Access Logging.
2. Delete **customer-created roles** that are no longer in use. If a role has Inline Policies or attached Managed Policies, follow the Console instructions to remove those permissions before deleting the role. **Do not delete shared AWS Managed Policies or service-linked roles.**
3. Review the resource lists in **CloudFront, S3, Lambda, API Gateway, DynamoDB, Cognito, EventBridge Scheduler, SNS, and CloudWatch**.
4. Open **Billing and Cost Management → Bills / Cost Explorer** to monitor charges after cleanup. Cost data may be delayed, and charges incurred before cleanup will still appear.

**Result:** The resources used by SecureTable have been stopped and deleted in the appropriate order, including automated schedules, the website, APIs, Lambda functions, databases, backups, monitoring resources, and IAM Roles that are no longer required.
