---
title : "Database Setup"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

In this section, we will create **Amazon DynamoDB** tables to store table information, bills, orders, menu items, ingredients, and operational history for **SecureTable**. The tables are deployed in **Asia Pacific (Singapore) – `ap-southeast-1`**, using **On-demand** capacity mode to accommodate demo traffic.

## 1. Create a DynamoDB Table

We will create the `SecureTable-Tables` table:

1. Open **Amazon DynamoDB** in the AWS Console and select **Tables → Create table**.
2. Enter **Table name:** `SecureTable-Tables`.
3. Enter **Partition key:** `table_id` with the **String** data type.
4. Leave **Sort key** empty.
5. Keep **Default settings** (**Capacity mode** defaults to **On-demand**).
6. Select **Create table** and wait until the table status changes to **Active**.

![DynamoDB Table Setup](../../images/5-Workshop/5.4-Database-setup/create-table.png)

**Note:** The `Partition key` and `Sort key` must match the **names and data types** used by the Lambda functions. Do not arbitrarily change `table_id` to `id`, as the backend would then be unable to retrieve the data.

## 2. Create the Remaining Tables

Repeat the steps above to create the remaining SecureTable tables. The following primary keys are based on the deployed data structure and source code:

| DynamoDB Table | Partition key | Sort key |
|---|---|---|
| `SecureTable-Tables` | `table_id` | — |
| `SecureTable-Bills` | `bill_id` | — |
| `SecureTable-Orders` | `order_id` | — |
| `SecureTable-OrderItems` | `item_id` | — |
| `SecureTable-MenuItems` | `dish_id` | — |
| `SecureTable-Ingredients` | `ingredient_id` | — |
| `SecureTable-InventoryReceipts` | `receipt_id` | — |
| `SecureTable-InventoryCorrections` | `receipt_id` | `correction_id` |
| `SecureTable-ReceiptCorrectionState` | `receipt_id` | — |
| `SecureTable-InventoryMovements` | `movement_id` | — |
| `SecureTable-InventoryCounts` | `count_id` | — |
| `SecureTable-FinanceLedger` | `entry_id` | — |
| `SecureTable-AuditLogs` | `audit_id` | — |
| `SecureTable-GuestSessions` | `session_hash` | — |
| `SecureTable-Idempotency` | `scope` | `key` |
| `SecureTable-PendingByIngredient` | `ingredient_id` | `item_id` |
| `SecureTable-NotificationOutbox` | `event_id` | — |
| `SecureTable-ReportDispatch` | `report_key` | — |

All keys listed above use the **String** data type. The system also uses `SecureTable-ReportDispatch` to track report delivery status and `SecureTable-NotificationOutbox` for notification events. These supporting tables must be created with the keys expected by their corresponding Lambda functions before those features are enabled.

![DynamoDB Tables](../../images/5-Workshop/5.4-Database-setup/table-list.png)

**Notes:**

- SecureTable uses a total of **18 DynamoDB tables**, configured with **On-demand** capacity mode.
- Of these, **13 business tables** are included in the scheduled backup scope.
- The `SecureTable-OrderItems` table uses **2 Global Secondary Indexes (GSIs)** to support queries.
- To create a GSI, navigate to **DynamoDB → Tables → SecureTable-OrderItems → Indexes → Create index** and configure it according to the system's query keys.
- Wait until the GSI status changes to **Active** before using it.

## 3. Enable Time to Live (TTL)

SecureTable uses TTL to automatically clean up temporary data.

1. Open the **`SecureTable-GuestSessions`** table.
2. Under **Additional settings**, find **Time to Live (TTL)** and select **Turn on**.
3. Enter **TTL attribute name:** `expires_at`, then save the changes.

![TTL Configuration](../../images/5-Workshop/5.4-Database-setup/ttl-config.png)

4. Repeat the process for the **`SecureTable-Idempotency`** table, using the `ttl` attribute.

**Note:** TTL values must be **Unix timestamps in seconds**, and DynamoDB deletes expired items asynchronously rather than immediately. Therefore, Lambda must still check session expiration when processing requests.

## 4. Add Initial Data

To allow Reception to open bills, table records must first be created in DynamoDB:

1. Open **`SecureTable-Tables` → Explore table items**.
2. Select **Create item**.
3. Add the following sample item:

```json
{
  "table_id": {
    "S": "T01"
  },
  "name": {
    "S": "Bàn 01"
  },
  "status": {
    "S": "FREE"
  }
}
```

4. Select **Create item**, then repeat for `T02`, `T03`, and so on, based on the number of tables in the restaurant.
5. Add menu and ingredient data to **`SecureTable-MenuItems`** and **`SecureTable-Ingredients`**.

![Table Items](../../images/5-Workshop/5.4-Database-setup/item-table.png)
![Menu Items](../../images/5-Workshop/5.4-Database-setup/item-menu.png)
![Ingredients](../../images/5-Workshop/5.4-Database-setup/item-ingre.png)

**Note:** **DynamoDB JSON** uses a different syntax from regular JSON. Make sure to select the correct input mode in the Console. The `dish_id` and `ingredient_id` values referenced in the menu and recipes must match the corresponding ingredient data.

## 5. Verify the Results

1. Navigate to **DynamoDB → Tables** and confirm that all created tables have the **Active** status.
2. Open **`SecureTable-Tables` → Explore table items** and verify the records for `T01`, `T02`, and so on.
3. Confirm that the tables use **On-demand** capacity mode.
4. Verify that **TTL** is enabled for the session and idempotency tables.

**Note:** **Point-in-Time Recovery (PITR)** must be enabled for the 13 tables included in the export-to-S3 backup scope. We will configure and verify PITR in [5.7 – Automated Reporting & Data Backup](5.7-Automation). PITR incurs separate charges, even when no export job is running.

Once these steps are complete, the database is ready to be connected to Lambda in [5.6 – Backend Development with AWS Lambda & API Gateway](5.6-Backend-dev).
