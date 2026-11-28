---
title : "End-to-End System Testing"
date : 2024-01-01
weight : 9
chapter : false
pre : " <b> 5.9. </b> "
---

After completing the main components of SecureTable, we will test several business workflows to verify that **CloudFront, Cognito, API Gateway, Lambda, and DynamoDB** work together. We will also test the automation features and the system's observability.

## 1. Test the Frontend and Authentication

1. Open `https://<cloudfront-domain>/` to verify that the website is delivered through **CloudFront** from **S3**.
2. Open the Staff login page and sign in with an **Amazon Cognito** account belonging to the `RECEPTION` or `KITCHEN` group. Confirm that the system redirects you to the appropriate interface.
3. Try calling a Staff API without a valid token; the request should be rejected. Then try accessing a function outside the signed-in account's permissions to verify authorization checks in Lambda.

**Expected result:** The website loads over HTTPS; Staff can sign in; API Gateway validates the token, and Lambda checks `cognito:groups` before allowing business operations.

## 2. Test the Ordering Workflow: Customer → Kitchen → Billing

1. On the **Reception** interface, select table `T01` and open a bill. Before opening the bill, scan T01's QR code to confirm that the Customer cannot place orders.
2. Scan the QR code after the bill has been opened, select a dish, and click **Send to Kitchen**. The system calls `POST /v1/guest/orders` through **CloudFront → API Gateway → Ordering Lambda**.
3. Open **DynamoDB** and check the new records in `SecureTable-Orders` and `SecureTable-OrderItems`, as well as the reserved ingredient quantities.
4. Switch to the **Kitchen** interface and select **Complete** or **Reject** for the newly placed item. Return to the Customer page and open **Ordered Items** to check its updated status.
5. On the **Reception** interface, close the bill and try placing another order using the old Guest Session. The system must reject requests associated with a closed bill.

**Expected result:** Orders flow from Customer to Kitchen, item statuses are updated in DynamoDB, ingredient quantities are handled according to whether an item is completed or rejected, and the bill cannot receive additional orders after being closed.

## 3. Test Inventory, Automation, and Monitoring

1. Sign in to the **Inventory** interface and create an ingredient purchase receipt. Check that the inventory quantity changes and confirm that a related entry appears in the **Audit Dashboard**, if that operation is recorded in the audit logs.
2. Open **Lambda `SecureTable-Report`** and wait for the scheduled time configured in EventBridge. Verify that the report email has been successfully delivered to the Admin's email account.
3. Open **Lambda `SecureTable-Backup`** and run a `dry_run` test to verify the backup prerequisites for the 13 DynamoDB tables. For an export initiated earlier, open **DynamoDB → Exports to S3** to confirm that its status is `COMPLETED`, then inspect the exported data in **S3**.
4. Open **CloudWatch Logs** for the Lambda function that was executed and **CloudTrail Event history** to review the corresponding execution information and AWS management activities.

After completing these tests, we can confirm that the main **SecureTable** workflows have been connected and tested across the user interface, APIs, database, and automation services.