---
title : "Creating and Configuring AWS Lambda Functions"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.6.2. </b> "
---

In this section, we will deploy **8 AWS Lambda Functions** for SecureTable using source code packages prepared on GitHub. Each function has a ZIP file containing its source code and a JSON file containing its corresponding **IAM Policy**. We will create each Lambda with the **default Execution Role**, then add the permissions required to access DynamoDB and other AWS services.

All Lambda functions will be created in **Asia Pacific (Singapore) – `ap-southeast-1`**, using **Python 3.15** (`python3.15`) for **Lambda Event Functions**.

## 1. Prepare the Source Code on GitHub

The source code has already been packaged in the SecureTable repository. Use the link below to download it before deployment:

**[Get the source code here](https://github.com/Akvlol/SecureTable-source-code/tree/main/Lambda)**

The `Lambda` directory has the following structure:

```text
Lambda/
├── GuestServices/
│   ├── SecureTable-GuestServices.zip
│   └── SecureTable-GuestServices.json
├── Ordering/
│   ├── SecureTable-Ordering.zip
│   └── SecureTable-Ordering.json
├── Reception/
│   ├── SecureTable-Reception.zip
│   └── SecureTable-Reception.json
├── Kitchen/
│   ├── SecureTable-Kitchen.zip
│   └── SecureTable-Kitchen.json
├── Inventory/
│   ├── SecureTable-Inventory.zip
│   └── SecureTable-Inventory.json
├── AdminDashboard/
│   ├── SecureTable-AdminDashboard.zip
│   └── SecureTable-AdminDashboard.json
├── Report/
│   ├── SecureTable-Report.zip
│   └── SecureTable-Report.json
└── Backup/
    ├── SecureTable-Backup.zip
    └── SecureTable-Backup.json
```

In each directory:

- The **`.zip`** file contains the source code package to upload to AWS Lambda.
- The **`.json`** file contains the IAM Policy used to grant permissions to that Lambda function's Execution Role.

You can download the entire repository by selecting **Code → Download ZIP** on GitHub, then extract the repository to locate the files in the `Lambda` directory. **Do not upload the ZIP file of the entire repository to Lambda**; use only the corresponding `SecureTable-<FunctionName>.zip` file.

**Note:** The IAM Policy files on GitHub are **configuration templates**. Before using them, replace `<YOUR_ACCOUNT_ID>` with the AWS Account ID of the account where you are deploying. Other placeholders, such as `<YOUR_BACKUP_BUCKET>`, must also be replaced with actual values if they appear in a policy.

## 2. Create an AWS Lambda Function

We will use `SecureTable-GuestServices` as an example. The remaining Lambda functions can be created in the same way.

1. Open **AWS Lambda → Functions → Create function** in the AWS Console.
2. Select **Event Function** if the interface asks you to choose a function type, then select **Author from scratch**.
3. Enter the following settings:

   | Setting | Value |
   |---|---|
   | **Function name** | `SecureTable-GuestServices` |
   | **Runtime** | `Python 3.15` |
   | **Architecture** | `x86_64` |
   | **Execution role** | `Create a new role with basic Lambda permissions` |

4. Select **Create function**.
5. Wait for AWS to create the function and open its details page.

![Create AWS Lambda](../../../images/5-Workshop/5.6-Backend/5.6.2-Lambda/create-lambda.png)

**Note:** When you select **Create a new role with basic Lambda permissions**, AWS automatically creates an **IAM Execution Role** and grants permissions to write logs to **Amazon CloudWatch** (through `AWSLambdaBasicExecutionRole`). Therefore, **you do not need to create an IAM Role manually before creating the Lambda function**. This role does not yet have permissions to access DynamoDB, S3, or SNS; we will add them in Section 4.

## 3. Upload Source Code to Lambda

1. In the repository, open the `Lambda/GuestServices` directory.
2. Download **`SecureTable-GuestServices.zip`** to your computer.
3. Return to the `SecureTable-GuestServices` function in AWS Lambda and open the **Code** tab.
4. Select **Upload from → .zip file**, choose the downloaded file, and confirm with **Save**.
5. Under **Runtime settings**, verify that the **Handler** is set to:

   ```text
   lambda_function.lambda_handler
   ```

6. If the Handler is incorrect, select **Edit**, enter the value above, and save the changes.

![Upload Lambda ZIP](../../../images/5-Workshop/5.6-Backend/5.6.2-Lambda/lambda-upload.png)

A typical ZIP package contains the main handler file, `lambda_function.py`, at its root. Some functions may include additional Python modules; for example, AdminDashboard uses modules for Finance and Audit Logs.

**Note:** Upload **the complete ZIP file for each Lambda function**. Do not extract it and upload individual files through the Console. Verify that `lambda_function.py` and any imported modules are placed correctly inside the ZIP package. Packages containing external libraries or code built for an older runtime should be tested for compatibility with Python 3.15.

## 4. Grant IAM Permissions to the Default Execution Role

After creating the Lambda function, AWS provides permissions for writing CloudWatch Logs but does not grant access to business data. We will use the **corresponding JSON file on GitHub** to create an Inline Policy for each function.

### 4.1. Prepare the IAM Policy

For `SecureTable-GuestServices`:

1. Open **`Lambda/GuestServices/SecureTable-GuestServices.json`** on GitHub.
2. Copy the entire JSON content.
3. Find and replace **every** occurrence of `<YOUR_ACCOUNT_ID>` with the **12-digit AWS Account ID** of the account you are using.
4. Verify that the DynamoDB table names and Region in the policy match the resources created in [Section 5.4](../5.4-Database-setup).

For example, a sample ARN:

```text
arn:aws:dynamodb:ap-southeast-1:<YOUR_ACCOUNT_ID>:table/SecureTable-Bills
```

After replacing the placeholder, the ARN must contain the actual Account ID. **IAM does not automatically replace `<YOUR_ACCOUNT_ID>`** when the policy is saved.

For **`SecureTable-Backup.json`**, also replace `<YOUR_BACKUP_BUCKET>` with the name of the S3 bucket used for backups. For **`SecureTable-Report.json`**, verify the ARN of the SNS Topic `SecureTable-FinancialReports`. These resources will be created and configured in [Section 5.7](../5.7-Automation).

### 4.2. Attach an Inline Policy to the Execution Role

1. Open **AWS Lambda → Functions → SecureTable-GuestServices**.
2. Select **Configuration → Permissions**.
3. Under **Execution role**, click the role name to open the **IAM Console**.
4. Select **Add permissions → Create inline policy**.
5. Switch to the **JSON** tab and paste the contents of `SecureTable-GuestServices.json` after replacing the Account ID.
6. Select **Next** and enter a policy name, such as `SecureTable-GuestServices-Policy`.
7. Select **Create policy** and verify that the new policy appears among the role's permissions.

![Lambda IAM Execution Role](../../../images/5-Workshop/5.6-Backend/5.6.2-Lambda/lambda-iam.png)

**Note:** Attach only the JSON file corresponding to each function to that function's Execution Role. **Do not** attach `AmazonDynamoDBFullAccess` to every Lambda function. The `AWSLambdaDynamoDBExecutionRole` policy is primarily intended for reading **DynamoDB Streams** and does not replace the table read/write permissions specified in SecureTable's JSON files.

## 5. Configure Memory, Timeout, and Environment Variables

1. Return to the Lambda function page and select **Configuration → General configuration → Edit**.
2. Adjust **Memory** and **Timeout** according to the function's requirements, then select **Save**.
3. If the source code requires environment variables, open **Configuration → Environment variables → Edit** and enter the appropriate values.
4. Confirm that the **Runtime** is `Python 3.15` and the **Handler** is `lambda_function.lambda_handler`.

![Lambda General Configuration](../../../images/5-Workshop/5.6-Backend/5.6.2-Lambda/lambda-general.png)

The following table provides **suggested initial settings** for a demo environment; verify and adjust them during deployment:

| Lambda Function | Suggested Memory | Suggested Timeout |
|---|---:|---:|
| `SecureTable-GuestServices` | 256 MB | 15 seconds |
| `SecureTable-Ordering` | 256 MB | 15 seconds |
| `SecureTable-Reception` | 512 MB | 30 seconds |
| `SecureTable-Kitchen` | 256 MB | 15 seconds |
| `SecureTable-Inventory` | 512 MB | 30 seconds |
| `SecureTable-AdminDashboard` | 512 MB | 30 seconds |
| `SecureTable-Report` | Depends on the task | Configure in [Section 5.7](../5.7-Automation) |
| `SecureTable-Backup` | Depends on the task | Configure in [Section 5.7](../5.7-Automation) |

**Note:** Memory and Timeout are not defined by the IAM Policy file. If a Lambda function reports a `Task timed out` error or runs out of memory, check CloudWatch Logs and adjust the configuration accordingly. For synchronous APIs, also consider the timeout limits of API Gateway.

## 6. Deploy the Remaining Lambda Functions

Repeat the process of **Create function → Upload ZIP → Configure → Create IAM Inline Policy** for all 8 Lambda functions:

| Lambda Function | Source Code File | IAM Policy File | Function |
|---|---|---|---|
| `SecureTable-GuestServices` | `SecureTable-GuestServices.zip` | `SecureTable-GuestServices.json` | Issue Guest Sessions and retrieve the menu |
| `SecureTable-Ordering` | `SecureTable-Ordering.zip` | `SecureTable-Ordering.json` | Place orders, reserve ingredients, and view ordered items |
| `SecureTable-Reception` | `SecureTable-Reception.zip` | `SecureTable-Reception.json` | Manage tables and open or close bills |
| `SecureTable-Kitchen` | `SecureTable-Kitchen.zip` | `SecureTable-Kitchen.json` | Manage the kitchen queue and complete or reject items |
| `SecureTable-Inventory` | `SecureTable-Inventory.zip` | `SecureTable-Inventory.json` | Record stock receipts, conduct inventory counts, and correct receipts |
| `SecureTable-AdminDashboard` | `SecureTable-AdminDashboard.zip` | `SecureTable-AdminDashboard.json` | View the Finance Dashboard and Audit Logs |
| `SecureTable-Report` | `SecureTable-Report.zip` | `SecureTable-Report.json` | Generate reports and send them through SNS |
| `SecureTable-Backup` | `SecureTable-Backup.zip` | `SecureTable-Backup.json` | Initiate DynamoDB Export to S3 |

The **`SecureTable-Report`** and **`SecureTable-Backup`** functions do not need API Gateway integration. They are triggered by **Amazon EventBridge Scheduler**; we will configure their schedules, SNS, and backups in [Section 5.7 – Automated Reporting & Data Backup](../5.7-Automation).

## 7. Verify the Results

1. Open **AWS Lambda → Functions** and confirm that all **8 Lambda Functions** have been created and that their names match the table above.

![List of SecureTable Lambda Functions](../../../images/5-Workshop/5.6-Backend/5.6.2-Lambda/lambda-list.png)

2. Open each function and verify that the correct ZIP package was uploaded, the **Runtime is Python 3.15**, and the **Handler is `lambda_function.lambda_handler`**.
3. Under **Configuration → Permissions**, verify that each function has an Execution Role and its corresponding Inline Policy, while retaining permissions to write CloudWatch Logs.
4. Confirm that all placeholders in the JSON files have been replaced, especially the Account ID, S3 bucket name, and SNS Topic.
5. You can use **Test** with an appropriate event to check for syntax, import, or configuration errors, then open **Monitor → View CloudWatch logs** when the function is invoked.

Once these steps are complete, all **8 SecureTable Lambda Functions** have been deployed using the ZIP files from GitHub, with **Python 3.15** and **default Execution Roles extended by function-specific IAM Policies**, ready to connect to API Gateway or scheduled tasks.
