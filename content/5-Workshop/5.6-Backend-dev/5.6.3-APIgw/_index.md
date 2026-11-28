---
title : "Integrating Lambda with Amazon API Gateway"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.6.3. </b> "
---

In this section, we will create a REST API, configure its **Resources and Methods**, set up a **Cognito Authorizer**, connect API Gateway to **CloudFront**, and then test the main features.

## 1. Create a REST API in Amazon API Gateway

1. Open **Amazon API Gateway** in the AWS Console and select the **Asia Pacific (Singapore) – `ap-southeast-1`** Region.
2. Select **APIs → Create API**.
3. Under **REST API**, choose **Build**.
4. Select **New API** and enter the following settings:

   | Setting | Value |
   |---|---|
   | **API name** | `SecureTable-API` |
   | **API type** | `REST API` |
   | **Endpoint Type** | `Regional` |

5. Select **Create API**, then open **Resources**.

![Create REST API](../../../images/5-Workshop/5.6-Backend/5.6.3-APIgw/create-api.png)

**Note:** SecureTable uses a **REST API**, not an HTTP API. All Lambda functions were created in `ap-southeast-1` in the previous section. The endpoints will begin with the `/v1` resource, while the **deployment stage** will be `dev`.

## 2. Create Resources and Methods for Guests

We will use `POST /v1/guest/sessions` as an example:

1. Under **Resources**, select the root resource `/` → **Create resource**.
2. Create a `v1` resource, then create `guest` under `/v1`, followed by `sessions` under `/v1/guest`.
3. Select `/v1/guest/sessions` → **Create method** → choose **POST** as the HTTP method.
4. Configure the integration as follows:
   - **Integration type:** `Lambda function`.
   - **Lambda proxy integration:** Enabled.
   - **Lambda function:** `SecureTable-GuestServices`.
   - **Region:** `ap-southeast-1`.
5. Under **Method request / Authorization**, select **NONE**.
6. Save the method and approve the permission for API Gateway to invoke Lambda if the Console displays a confirmation prompt.

Continue creating the remaining Guest endpoints:

| Method | Resource | Lambda integration | Authorization |
|---|---|---|---|
| `POST` | `/v1/guest/sessions` | `SecureTable-GuestServices` | `NONE` |
| `GET` | `/v1/guest/menu` | `SecureTable-GuestServices` | `NONE` |
| `POST` | `/v1/guest/orders` | `SecureTable-Ordering` | `NONE` |
| `GET` | `/v1/guest/orders` | `SecureTable-Ordering` | `NONE` |

![Guest API Resources](../../../images/5-Workshop/5.6-Backend/5.6.3-APIgw/guest-routes.png)

**Note:** `Authorization: NONE` only means that **API Gateway does not use a Cognito Authorizer** for these methods. `POST /guest/sessions` still requires the table to have an open bill; the other Guest APIs require `Authorization: Bearer <guest_session_token>`, which is validated by Lambda. In particular, `GET /guest/orders` only returns items belonging to the open bill associated with the Guest Session; it does not use a `bill_id` provided by the customer.

## 3. Create a Cognito User Pool Authorizer for Staff

1. In `SecureTable-API`, open **Authorizers → Create authorizer**.
2. Enter **Authorizer name:** `SecureTableStaffAuthorizer`.
3. Select **Authorizer type:** `Cognito` / `Cognito user pool` (depending on the Console interface).
4. Select the **Cognito User Pool** created for Staff in `ap-southeast-1`.
5. Under **Token source**, enter `Authorization`.
6. Select **Create authorizer**.

![Cognito Authorizer](../../../images/5-Workshop/5.6-Backend/5.6.3-APIgw/cognito-authorizer.png)

When creating a Staff method, select `SecureTableStaffAuthorizer` under **Method request → Authorization** and **leave Authorization Scopes empty**.

**Note:** The SecureTable frontend sends a **Cognito ID Token** when calling Staff APIs. With a REST API Cognito User Pool Authorizer, leaving **Authorization Scopes** empty allows API Gateway to validate the ID Token. Lambda then checks `cognito:groups` to determine whether the user has the `RECEPTION`, `KITCHEN`, `INVENTORY`, or `ADMIN` role. Successfully signing in does not automatically grant access to every endpoint.

## 4. Integrate Staff APIs with Lambda

1. Return to **Resources** and create a `staff` resource under `/v1`.
2. Create child resources for each business function, such as `bills`, `kitchen`, `inventory`, and `finance`.
3. Create the HTTP methods for each resource and enable **Lambda proxy integration**.
4. Select the Lambda function according to the table below.
5. For **all Staff Methods**, set **Authorization:** `SecureTableStaffAuthorizer` and leave **Authorization Scopes** empty.

| Method | Resource | Lambda integration |
|---|---|---|
| `GET` | `/v1/staff/tables` | `SecureTable-Reception` |
| `GET`, `POST` | `/v1/staff/bills` | `SecureTable-Reception` |
| `GET` | `/v1/staff/bills/{billId}` | `SecureTable-Reception` |
| `GET` | `/v1/staff/bills/{billId}/receipt` | `SecureTable-Reception` |
| `POST` | `/v1/staff/bills/{billId}/close` | `SecureTable-Reception` |
| `GET` | `/v1/staff/kitchen/queue` | `SecureTable-Kitchen` |
| `POST` | `/v1/staff/kitchen/items/{itemId}/complete` | `SecureTable-Kitchen` |
| `POST` | `/v1/staff/kitchen/items/{itemId}/reject` | `SecureTable-Kitchen` |
| `GET`, `POST` | `/v1/staff/inventory/receipts` | `SecureTable-Inventory` |
| `GET`, `POST` | `/v1/staff/inventory/receipts/{receiptId}/corrections` | `SecureTable-Inventory` |
| `GET` | `/v1/staff/finance/daily/{date}` | `SecureTable-AdminDashboard` |
| `GET` | `/v1/staff/finance/summary` | `SecureTable-AdminDashboard` |
| `GET` | `/v1/staff/finance/weekly` | `SecureTable-AdminDashboard` |
| `GET` | `/v1/staff/audit-logs` | `SecureTable-AdminDashboard` |

**Notes:**

- When creating `{billId}`, `{itemId}`, `{receiptId}`, and `{date}`, preserve the curly braces and exact capitalization. These are **Path Parameters** passed to Lambda through `pathParameters`.
- For resources with multiple methods, such as `/v1/staff/bills`, create **GET** and **POST** separately, even if both invoke the same Lambda function.
- The table above lists the main endpoints in the SecureTable design. If the deployed `Inventory` version includes additional endpoints for viewing ingredients, receipt details, or inventory counts, create the corresponding Methods and Resources according to the actual Lambda code and frontend requests. Do not create routes that the Lambda does not implement.

## 5. Deploy the REST API to the `dev` Stage

After creating the Resources, Methods, and Authorizer:

1. On the `SecureTable-API` page, select **Deploy API**.
2. Under **Stage**, select **New stage** if this is the first deployment.
3. Enter **Stage name:** `dev`, then confirm **Deploy**.
4. Open **Stages → dev** and copy the **Invoke URL**.

The Invoke URL has the following format:

```text
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/dev
```

For example, the full URL for calling the Guest Session API is:

```text
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/dev/v1/guest/sessions
```

![Deploy API](../../../images/5-Workshop/5.6-Backend/5.6.3-APIgw/deply-api.png)

**Note:** `dev` is the **Stage**, while `v1` is a **Resource**. Every time you create or modify a Method, Integration, or Authorization setting in a REST API, you must **deploy the API again to the `dev` stage** for the changes to take effect. Otherwise, users may still be calling the previous API deployment.

## 6. Connect API Gateway to Amazon CloudFront

In the source code, the frontend calls APIs using relative paths such as `fetch('/v1/guest/menu')`. Therefore, CloudFront must be configured to forward `/v1/*` requests to API Gateway.

### 6.1. Add an API Gateway Origin

1. Open **CloudFront → Distributions** and select the distribution serving the SecureTable website.
2. Select **Origins → Create origin**.
3. Configure the following settings:

   | Setting | Value |
   |---|---|
   | **Origin domain** | `<api-id>.execute-api.ap-southeast-1.amazonaws.com` |
   | **Origin path** | `/dev` |
   | **Protocol** | `HTTPS only` |

4. Save the newly created Origin.

![CloudFront API Origin](../../../images/5-Workshop/5.6-Backend/5.6.3-APIgw/cloudfront-origin.png)

### 6.2. Create a CloudFront Behavior for the API

1. In the same distribution, open **Behaviors → Create behavior**.
2. Apply the following settings:

   | Setting | Value |
   |---|---|
   | **Path pattern** | `/v1/*` |
   | **Origin** | The API Gateway Origin created above |
   | **Viewer protocol policy** | `Redirect HTTP to HTTPS` |
   | **Allowed HTTP methods** | `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE` |
   | **Cache policy** | `CachingDisabled` |
   | **Origin request policy** | `AllViewerExceptHostHeader` |

3. Save the Behavior and confirm that the `/v1/*` behavior has the appropriate priority ahead of `Default (*)`.
4. Verify that the **Default behavior (`*`)** still uses the S3 Origin with OAC to serve HTML, CSS, and JavaScript.

![CloudFront API Behavior](../../../images/5-Workshop/5.6-Backend/5.6.3-APIgw/cloudfront-behavior.png)

When the frontend requests:

```text
https://<cloudfront-domain>/v1/guest/menu
```

CloudFront forwards the request to API Gateway at:

```text
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/dev/v1/guest/menu
```

**Important notes:**

- **Set the Origin path to `/dev` only**, not `/dev/v1`. Because the browser already requests `/v1/...`, adding `v1` to the Origin path would duplicate the path and could cause `403`/`404` errors.
- **Do not include `https://`** in the Origin domain field.
- `CachingDisabled` prevents CloudFront from caching dynamic API responses. `AllViewerExceptHostHeader` forwards the necessary headers, including `Authorization`, while using the API Gateway Origin's Host header. This configuration is suitable for API Gateway endpoints that use Bearer Tokens.
- Because the frontend and API share the same CloudFront domain, standard `fetch('/v1/...')` requests **do not require separate CORS configuration**. If the frontend later calls the `execute-api` domain directly, CORS must be configured appropriately.

After completing these steps, **the SecureTable frontend is connected to AWS Lambda through CloudFront and API Gateway**, and the Staff APIs are authenticated through Cognito. In the next section, [**5.7 – Automated Reporting & Data Backup**](../../5.7-Automation), we will configure EventBridge Scheduler, Amazon SNS, and the process of exporting DynamoDB backups to Amazon S3.
