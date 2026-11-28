---
title: "Workshop"
date: 2026-11-23
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Building the SecureTable Restaurant Management System with AWS Serverless

#### Overview

**AWS Serverless** enables developers to build and deploy applications without managing servers directly. Services such as AWS Lambda, Amazon API Gateway, and Amazon DynamoDB handle requests, store data, and scale according to demand, reducing operational overhead.

In this workshop, we will learn how to build and deploy **SecureTable**, a restaurant management system on AWS that focuses on ingredient inventory management, QR code ordering, order processing, bill management, and revenue tracking.

The system uses a Serverless architecture, combining AWS services to deliver the web interface, process business logic, authenticate users, store data, and monitor system operations.

The main components we will deploy include:

- **Frontend – Amazon S3 & Amazon CloudFront:** Host and deliver web interfaces for customers and staff over HTTPS.
- **Authentication – Amazon Cognito & AWS IAM:** Authenticate staff, manage permissions, and control access to AWS resources.
- **Backend – AWS Lambda & Amazon API Gateway:** Build APIs for ordering, table and bill management, kitchen operations, inventory management, and reporting.
- **Database – Amazon DynamoDB:** Store restaurant data and use DynamoDB Transactions for operations that require data consistency.
- **Automation – Amazon EventBridge Scheduler & Amazon SNS:** Schedule reports and data backups, and send email notifications.
- **Monitoring & Security – Amazon CloudWatch, AWS CloudTrail & AWS WAF:** Monitor the system, record AWS activity, and provide additional web protection when WAF is enabled.

By the end of this workshop, we will have a working SecureTable system deployed on AWS: customers can scan QR codes to place orders, staff can manage restaurant operations, and administrators can view operational data.

The workshop also covers testing, monitoring, and resource optimization based on the principles of the **AWS Well-Architected Framework**.


#### Content

1. [System Overview & Architecture](5.1-System-overview/)
2. [Prerequisites](5.2-Prerequisite/)
3. [Frontend Deployment](5.3-Frontend-dev/)
4. [Database Setup](5.4-Database-setup/)
5. [Authentication & Authorization](5.5-Auth/)
6. [Backend Development](5.6-Backend-dev/)
7. [Automated Reporting & Data Backup](5.7-Automation/)
8. [Security Hardening & System Monitoring](5.8-Security/)
9. [End-to-End System Testing](5.9-E2E-testing/)
10. [Resource Cleanup](5.10-Cleanup/)
