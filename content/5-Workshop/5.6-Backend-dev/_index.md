---
title : "Backend Development"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

The system uses a **Serverless** architecture: there is no need to manage EC2 instances or continuously running web servers. Lambda functions are invoked when API requests arrive or scheduled events occur.

## 1. Backend Processing Architecture

There are two main workflows:

1. **Guest:** Customer scans a QR code → CloudFront → API Gateway → GuestServices/Ordering Lambda → DynamoDB. Guests do not sign in through Cognito; the Lambda functions validate the **Guest Session Token** and the bill status.
2. **Staff:** Staff sign in through Cognito → CloudFront → API Gateway (**Cognito Authorizer**) → Corresponding Lambda → DynamoDB. In addition to authentication, the Lambda functions check whether the user belongs to the `RECEPTION`, `KITCHEN`, `INVENTORY`, or `ADMIN` group.

Requests from the frontend use paths under `/v1/*`. In CloudFront, these requests are routed to the **API Gateway REST API** instead of S3.

## 2. SecureTable Lambda Functions

After consolidating functions that handle related business operations, SecureTable organizes its backend into **8 Lambda functions**:

| Lambda | Main Responsibility | Trigger |
|---|---|---|
| `SecureTable-GuestServices` | Issue Guest Sessions and return the menu | API Gateway – Guest |
| `SecureTable-Ordering` | Receive orders, reserve ingredients, and retrieve previously ordered items | API Gateway – Guest |
| `SecureTable-Reception` | View tables, open/close bills, and view bill details | API Gateway – Staff |
| `SecureTable-Kitchen` | View the kitchen queue, complete or reject ordered items | API Gateway – Staff |
| `SecureTable-Inventory` | Record stock receipts, view receipt records, and adjust inventory | API Gateway – Staff |
| `SecureTable-AdminDashboard` | Aggregate financial data and query Audit Logs | API Gateway – Staff |
| `SecureTable-Report` | Generate and send scheduled reports | EventBridge Scheduler |
| `SecureTable-Backup` | Initiate DynamoDB backups to S3 | EventBridge Scheduler |

**Note:** `Report` and `Backup` do not require public API endpoints. EventBridge schedules, SNS configuration, and S3 backups will be covered in [5.7 – Automated Reporting & Data Backup](5.7-Automation).

## 3. Deployment Order

We will proceed through three sections:

1. [5.6.1 – Backend Overview & API Design](5.6-Backend-dev/5.6.1-Overview): Identify the endpoints required by Guests and Staff, their HTTP methods, and the Lambda functions that handle them.
2. [5.6.2 – Creating and Configuring AWS Lambda Functions](5.6-Backend-dev/5.6.2-Lambda): Create functions, upload source code, configure runtimes, and assign IAM Execution Roles.
3. [5.6.3 – Integrating Lambda with Amazon API Gateway](5.6-Backend-dev/5.6.3-APIgw): Create a REST API, configure Lambda proxy integration, enable the Cognito Authorizer, and configure CloudFront so the frontend can call the API.

**Note:** DynamoDB tables must be created with the correct names and keys in Section 5.4 before testing the Lambda functions. In addition, the frontend can call the API only after API Gateway has been deployed and the `/v1/*` CloudFront behavior has been configured correctly.