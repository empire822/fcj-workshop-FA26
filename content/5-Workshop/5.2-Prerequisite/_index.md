---
title : "Prerequiste"
date : 2024-01-01 
weight : 2 
chapter : false
pre : " <b> 5.2. </b> "
---

SecureTable was built and deployed using an **AWS Free Plan** account, which is available for up to six months. However, some AWS services may be restricted or require additional account verification before they can be used.

During the deployment of SecureTable, you may encounter this issue when creating an **Amazon CloudFront distribution**.

## 1. Check CloudFront Access

Before starting the workshop, verify that your AWS account can create a CloudFront distribution.

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/).
2. Search for and open **CloudFront**.
3. Select **Create distribution** and proceed through the configuration steps.
4. Check whether AWS allows you to create the distribution.

If your account has not been verified, AWS may display the following error message:

> Your account must be verified before you can add new CloudFront resources. To verify your account, please contact AWS Support (https://console.aws.amazon.com/support/home#/) and include this error message.

If you encounter this message, contact AWS Support to request account verification.

## 2. Request Account Verification Through AWS Support

To request access to CloudFront, follow these steps:

1. Open the [AWS Support Center](https://console.aws.amazon.com/support/home#/).
2. Select **Create case**.
3. Choose **Account and billing** as the case type.
4. Select **Account Activation → Account Verification**, if these options are available.
5. Enter a subject, such as `CloudFront Account Verification Request`.
6. Describe the issue and include the complete CloudFront error message.
7. Select **Submit** to send the support request.

After receiving the request, AWS Support will review your account and determine whether CloudFront access can be enabled.

Based on our deployment experience, AWS may respond within approximately **one business day** and complete the verification process within **1–2 business days**. Actual processing times may vary depending on the account.

Once AWS Support confirms that the restriction has been removed, return to CloudFront and try creating the distribution again.

**Notes:**

- We are not sure if all AWS Free Plan accounts encounter this restriction.
- Processing times depend on AWS's verification process and are not guaranteed.
- If your account can already create CloudFront distributions, you can skip this step and continue with the workshop.
