---
title : "Backend Overview & API Design"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.6.1. </b> "
---

## 1. API Request Processing Flow

1. The browser sends an HTTPS request to `https://<cloudfront-domain>/v1/...`.
2. CloudFront forwards requests matching `/v1/*` to API Gateway.
3. API Gateway matches the corresponding **Resource + Method**; for Staff APIs, the **Cognito User Pool Authorizer** validates the token.
4. API Gateway invokes Lambda using **Lambda proxy integration**.
5. Lambda validates the request data and business permissions, accesses DynamoDB, and returns JSON with an HTTP status code.

## 2. Guest API Design

Guests do not use Cognito. The session creation endpoint can be accessed without a token; the remaining Guest endpoints require a token issued by the backend.

| Method | Endpoint | Lambda | Function |
|---|---|---|---|
| `POST` | `/v1/guest/sessions` | `SecureTable-GuestServices` | Validate the table/bill and issue a Guest Session |
| `GET` | `/v1/guest/menu` | `SecureTable-GuestServices` | Return the list of dishes available for ordering |
| `POST` | `/v1/guest/orders` | `SecureTable-Ordering` | Place an order and reserve ingredients |
| `GET` | `/v1/guest/orders` | `SecureTable-Ordering` | View ordered items and their statuses in the current bill |

**Example: Creating a session for table `T01`:**

```http
POST /v1/guest/sessions
Content-Type: application/json
```

```json
{
  "table_id": "T01"
}
```

Lambda checks that the table is `OCCUPIED` and the bill is `OPEN`. If valid, the API returns a `guest_session_token`. When calling the order API, the frontend sends the token in the header:

```http
Authorization: Bearer <guest_session_token>
Content-Type: application/json
Idempotency-Key: <unique-key-for-this-order>
```

Example order request body (the dish ID must exist in `SecureTable-MenuItems`):

```json
{
  "items": [
    { "dish_id": "D_BUN_BO_HUE", "qty": 2 }
  ]
}
```

After placing an order, guests can call `GET /v1/guest/orders` to view items with the statuses `ORDERED`, `COMPLETED`, or `REJECTED`, even after reloading the page. **Guests do not have an endpoint for canceling orders**.

**Note:** Lambda determines the `bill_id` from the Guest Session rather than accepting it as a user-supplied parameter. Guests can only read items belonging to the open bill associated with that session.

## 3. Staff API Design

Staff members sign in through Cognito before calling endpoints under `/v1/staff/*`.

| Method | Endpoint | Lambda | Authorized Groups |
|---|---|---|---|
| `GET` | `/v1/staff/tables` | `SecureTable-Reception` | `RECEPTION`, `ADMIN` |
| `POST` | `/v1/staff/bills` | `SecureTable-Reception` | `RECEPTION`, `ADMIN` |
| `GET` | `/v1/staff/bills` | `SecureTable-Reception` | `RECEPTION`, `ADMIN` |
| `GET` | `/v1/staff/bills/{billId}` | `SecureTable-Reception` | `RECEPTION`, `ADMIN` |
| `GET` | `/v1/staff/bills/{billId}/receipt` | `SecureTable-Reception` | `RECEPTION`, `ADMIN` |
| `POST` | `/v1/staff/bills/{billId}/close` | `SecureTable-Reception` | `RECEPTION`, `ADMIN` |
| `GET` | `/v1/staff/kitchen/queue` | `SecureTable-Kitchen` | `KITCHEN`, `ADMIN` |
| `POST` | `/v1/staff/kitchen/items/{itemId}/complete` | `SecureTable-Kitchen` | `KITCHEN`, `ADMIN` |
| `POST` | `/v1/staff/kitchen/items/{itemId}/reject` | `SecureTable-Kitchen` | `KITCHEN`, `ADMIN` |
| `GET`, `POST` | `/v1/staff/inventory/receipts` | `SecureTable-Inventory` | `INVENTORY`, `ADMIN` |
| `GET`, `POST` | `/v1/staff/inventory/receipts/{receiptId}/corrections` | `SecureTable-Inventory` | `INVENTORY`, `ADMIN` |
| `GET` | `/v1/staff/finance/daily/{date}` | `SecureTable-AdminDashboard` | `ADMIN` |
| `GET` | `/v1/staff/finance/summary` | `SecureTable-AdminDashboard` | `ADMIN` |
| `GET` | `/v1/staff/finance/weekly` | `SecureTable-AdminDashboard` | `ADMIN` |
| `GET` | `/v1/staff/audit-logs` | `SecureTable-AdminDashboard` | `ADMIN` |

**Note:** This table shows the main routes found in SecureTable's source code and deployment documentation. If your version of `Inventory` includes additional routes for reading ingredients, viewing receipt details, or making separate adjustments, create the exact Methods/Resources used by that frontend and Lambda version. Do not create routes solely because they appear in the original proposed API Spec.

**Example: Opening a bill:**

```http
POST /v1/staff/bills
Authorization: Bearer <staff_id_token>
Content-Type: application/json
Idempotency-Key: <unique-key-for-this-open-bill>
```

```json
{
  "table_id": "T01",
  "guest_count": 2
}
```

API Gateway validates the Cognito ID Token. The Reception Lambda then checks the user's group, `guest_count`, `Idempotency-Key`, and whether the table already has an open bill. If the table is occupied, the backend may return **`409 Conflict`**.

## 4. Guest vs. Staff Authentication

| Item | Guest | Staff |
|---|---|---|
| Cognito sign-in | No | Yes |
| Token | Guest Session Token | Cognito ID Token |
| Validation at API Gateway | `Authorization: NONE` | Cognito User Pool Authorizer |
| Validation in Lambda | Session expiration, correct table, and open bill | `sub`, `cognito:groups`, and business permissions |
| Access scope | Menu, order placement, and items in the bill | Based on staff group |

For SecureTable's REST API and Cognito User Pool Authorizer, Staff Methods use an **ID Token** and leave **Authorization Scopes** empty. If scopes are required, API Gateway expects an access token containing the corresponding scopes, which does not match how the current frontend operates.

## 5. Data and Error Conventions

- **Request/Response:** JSON, `Content-Type: application/json`.
- **Status codes:** `200` for successful reads/updates, `201` for creation, `400` for invalid data, `401` for missing/invalid authentication, `403` for insufficient permissions, `409` for state conflicts, and `500` for server errors.
- **Orders:** Use an `Idempotency-Key` for `POST /guest/orders` to prevent duplicate orders if a customer retries after a network issue.
- **DynamoDB:** Operations affecting multiple tables use transactions/conditional writes to prevent inventory from being deducted or charges from being added twice.
- **Security:** Do not trust client-supplied `role`, `bill_id`, or `table_id` for operations requiring authorization checks; do not log tokens to CloudWatch Logs.