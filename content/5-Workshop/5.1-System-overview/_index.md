---
title : "System Overview & Architecture"
date : 2024-01-01 
weight : 1 
chapter : false
pre : " <b> 5.1. </b> "
---

Before deploying services through the AWS Console, we need to understand how the components of **SecureTable** connect and exchange data. The system is built on an **AWS Serverless** architecture, focusing on three key requirements: supporting restaurant operations, protecting data, and optimizing costs for a small-scale deployment.

In this section, we will examine the architecture diagram used to deploy SecureTable, its main workflows, and the technologies selected for the project.

## 1. SecureTable Architecture Diagram

![SecureTable AWS Architecture](../../images/2-Proposal/SecureTable_Architecture.svg)

### 1.1. Guest Workflow

1. **Accessing the website:** Customers scan the static QR code at their table. Their browser loads the interface through **Amazon CloudFront**, with the HTML, CSS, and JavaScript files stored in **Amazon S3**.
2. **Creating a Guest Session:** The frontend sends `POST /v1/guest/sessions` to **Amazon API Gateway**. The **GuestServices Lambda** checks in **Amazon DynamoDB** that the table is `OCCUPIED` and its associated bill is `OPEN`. If both conditions are met, the system issues a Guest Session Token.
3. **Viewing the menu:** The frontend calls `GET /v1/guest/menu` with the token. GuestServices validates the session, retrieves menu items and ingredient availability from DynamoDB, and returns a list of dishes available for ordering.
4. **Placing an order:** The frontend sends `POST /v1/guest/orders` to the **Ordering Lambda**. The function validates the session and order data, then performs **DynamoDB Transactions** to record the order and reserve the required ingredients.
5. **Order processing:** Ordered items are stored in the system for the Kitchen team to process. Customers can place additional orders while their bill remains open.

Customers **do not sign in through Amazon Cognito**. Access to Guest APIs is controlled by the Guest Session Token and the bill status checked by the backend.

### 1.2. Staff Workflow

1. **Signing in:** Staff members access the Staff interface through CloudFront and authenticate using **Amazon Cognito User Pool**.
2. **Authorization:** After signing in, staff members are redirected to the interface associated with their role. Staff APIs are protected by a **Cognito Authorizer** in API Gateway, with additional permission checks in Lambda.
3. **Reception:** The **Reception Lambda** manages tables, opens bills, displays bill details, and closes bills. During bill closure, the system first changes the bill status to `CLOSING` before finalizing the process and releasing the table.
4. **Kitchen:** The **Kitchen Lambda** retrieves pending items and allows staff to complete or reject them. Changes affecting inventory and bills are handled according to the business rules.
5. **Inventory:** The **Inventory Lambda** records ingredient purchases, updates stock quantities, and stores adjustment history.
6. **Admin:** The **AdminDashboard Lambda** provides financial data and activity records (**Audit Logs**) to support reviews of restaurant operations.

Each Lambda accesses the required DynamoDB tables using its **IAM Execution Role**. Critical write operations use conditional updates, transactions, or **Idempotency Keys** to reduce conflicts and prevent duplicate processing.

### 1.3. Automated Reporting and Backup

- **Reporting:** **Amazon EventBridge Scheduler** invokes the **Report Lambda** on a schedule. The Lambda aggregates data from DynamoDB and sends reports to the administrator's email through **Amazon SNS**.
- **Backup:** EventBridge Scheduler invokes the **Backup Lambda**. The Lambda initiates **DynamoDB Export to S3** for the tables included in the backup scope. The data is stored in S3, and **S3 Lifecycle** can transition backup copies to **S3 Glacier Flexible Retrieval** after the configured period.

These two workflows operate independently of regular ordering requests. Data exports and storage-class transitions are asynchronous operations and do not provide instant recovery.

### 1.4. Monitoring, Security, and Reliability

- **Amazon CloudWatch:** Collects Lambda logs, metrics, and API Gateway monitoring data. CloudWatch Alarms can send alerts through SNS.
- **AWS CloudTrail:** Helps track AWS API activity at the account level. CloudTrail is separate from the business-level Audit Logs stored in DynamoDB.
- **Amazon Cognito & AWS IAM:** Authenticate staff and control access to AWS resources according to the **principle of least privilege**.
- **Amazon CloudFront & HTTPS:** Deliver the website over encrypted connections. **AWS WAF** provides additional request filtering on the CloudFront distribution.
- **DynamoDB Transactions & Backup:** Help maintain data consistency for critical operations and provide data for recovery in the event of a failure.

**Security limitation:** Static QR codes can be copied. Guest Sessions are linked to open bills, so the current mechanism does not verify whether the person scanning the QR code is physically present at the table.

---

## 2. Technology Stack

| Component | Technologies / Services |
|---|---|
| **Frontend** | HTML, CSS, JavaScript; Amazon S3, Amazon CloudFront |
| **Backend** | Python, AWS Lambda, Amazon API Gateway (REST API) |
| **Database** | Amazon DynamoDB |
| **Authentication & Authorization** | Amazon Cognito, AWS IAM, Guest Session Token |
| **Automation & Notifications** | Amazon EventBridge Scheduler, Amazon SNS |
| **Backup & Storage** | DynamoDB Export to S3, Amazon S3, S3 Glacier Flexible Retrieval |
| **Monitoring & Security** | Amazon CloudWatch, AWS CloudTrail; AWS WAF |
| **Deployment** | AWS Management Console |

---

## 3. Reference Budget for the Demo Environment

In the Proposal, we estimated the budget for a restaurant with **10 tables**, approximately **20–30 bills per day**, **40,000 REST API requests per month**, **60,000 Lambda invocations per month**, and approximately **1 GB of DynamoDB data**, deployed in the Singapore Region (`ap-southeast-1`).

### 3.1. Estimated Costs for the Demo Configuration

| No. | Service / Cost Category | Estimated Cost (USD/month) |
|:---:|---|---:|
| 1 | Amazon CloudFront & S3 Frontend | $0.50 |
| 2 | Amazon API Gateway (REST) | $0.20 |
| 3 | AWS Lambda | $0.30 |
| 4 | Amazon DynamoDB | $1.00 |
| 5 | Amazon Cognito | $0.00 |
| 6 | Amazon CloudWatch | $1.00 |
| 7 | Amazon EventBridge Scheduler & SNS | $0.10 |
| 8 | AWS WAF (included in the CloudFront Free Plan) | $0.00 |
| 9 | DynamoDB PITR, Export & S3 Backup | $2.50 |
| 10 | AWS CloudTrail Event History | $0.00 |
| | **Total Service Cost** | **$5.60** |
| | **Contingency Allowance** | **$2.00** |
| | **Total Estimated Budget** | **$7.60/month** |

**Note:** The $0 WAF estimate applies only if the distribution uses the **CloudFront Free Plan** and WAF is included in the plan. Lambda, PITR, export, and storage costs depend on the actual configuration and data volume.

### 3.2. Cost Optimization Measures Implemented

- Adopted a **Serverless** architecture instead of maintaining a continuously running EC2 server.
- Organized Lambda functions by business capability; for example, **Reception** combines table and bill management.
- Adjusted Lambda **Memory** and **Timeout** based on CloudWatch metrics.
- Delivered the static frontend through CloudFront and S3 without a separate web server.
- Used S3 Lifecycle to manage long-term backup storage.
