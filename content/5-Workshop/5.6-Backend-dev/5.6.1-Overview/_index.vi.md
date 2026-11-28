---
title : "Tổng quan Backend & Thiết kế API"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.6.1. </b> "
---

## 1. Luồng xử lý một API request

1. Trình duyệt gửi request HTTPS đến `https://<cloudfront-domain>/v1/...`.
2. CloudFront chuyển request `/v1/*` đến API Gateway.
3. API Gateway tìm **Resource + Method** tương ứng; với Staff API, **Cognito User Pool Authorizer** kiểm tra token.
4. API Gateway gọi Lambda bằng **Lambda proxy integration**.
5. Lambda xác minh dữ liệu và quyền nghiệp vụ, truy cập DynamoDB, rồi trả về JSON cùng HTTP status code.

## 2. Thiết kế Guest API

Guest không dùng Cognito. Endpoint tạo phiên được truy cập khi chưa có token; các endpoint Guest còn lại yêu cầu token do backend cấp.

| Method | Endpoint | Lambda | Chức năng |
|---|---|---|---|
| `POST` | `/v1/guest/sessions` | `SecureTable-GuestServices` | Kiểm tra bàn/bill và cấp Guest Session |
| `GET` | `/v1/guest/menu` | `SecureTable-GuestServices` | Trả danh sách món có thể đặt |
| `POST` | `/v1/guest/orders` | `SecureTable-Ordering` | Đặt món và giữ nguyên liệu |
| `GET` | `/v1/guest/orders` | `SecureTable-Ordering` | Xem món và trạng thái trong bill hiện tại |

**Ví dụ tạo phiên cho bàn `T01`:**

```http
POST /v1/guest/sessions
Content-Type: application/json
```

```json
{
  "table_id": "T01"
}
```

Lambda kiểm tra bàn `OCCUPIED` và bill `OPEN`. Nếu hợp lệ, API trả `guest_session_token`. Khi gọi API đặt món, frontend gửi token trong header:

```http
Authorization: Bearer <guest_session_token>
Content-Type: application/json
Idempotency-Key: <unique-key-for-this-order>
```

Ví dụ body đặt món (mã món phải tồn tại trong `SecureTable-MenuItems`):

```json
{
  "items": [
    { "dish_id": "D_BUN_BO_HUE", "qty": 2 }
  ]
}
```

Sau khi gửi, khách có thể gọi `GET /v1/guest/orders` để xem món `ORDERED`, `COMPLETED` hoặc `REJECTED`, kể cả khi tải lại trang. **Khách không có endpoint hủy món**.

**Lưu ý:** `bill_id` được Lambda xác định từ Guest Session, không lấy từ tham số do người dùng nhập. Guest chỉ đọc các món thuộc bill đang mở gắn với session đó.

## 3. Thiết kế Staff API

Nhân viên đăng nhập qua Cognito trước khi gọi các endpoint `/v1/staff/*`.

| Method | Endpoint | Lambda | Nhóm quyền |
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

**Lưu ý:** Bảng thể hiện những route chính trong mã nguồn/tài liệu triển khai SecureTable. Nếu phiên bản `Inventory` của m có thêm route đọc nguyên liệu, xem chi tiết phiếu nhập hoặc điều chỉnh riêng, hãy tạo thêm đúng các Method/Resource mà frontend và Lambda phiên bản đó đang dùng; không tạo route chỉ vì nó có trong bản API Spec đề xuất ban đầu.

**Ví dụ mở bill:**

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

API Gateway xác minh ID Token từ Cognito, sau đó Lambda Reception kiểm tra nhóm quyền, `guest_count` và `Idempotency-Key`, cùng điều kiện bàn chưa có bill mở. Nếu bàn đang được sử dụng, backend có thể trả về **`409 Conflict`**.

## 4. Phân biệt xác thực Guest và Staff

| Nội dung | Guest | Staff |
|---|---|---|
| Đăng nhập Cognito | Không | Có |
| Token | Guest Session Token | Cognito ID Token |
| Kiểm tra tại API Gateway | `Authorization: NONE` | Cognito User Pool Authorizer |
| Kiểm tra tại Lambda | Session còn hạn, đúng bàn và bill mở | `sub`, `cognito:groups` và quyền nghiệp vụ |
| Phạm vi | Menu, đặt món, xem món của bill | Theo nhóm nhân viên |

Với REST API và Cognito User Pool Authorizer của SecureTable, các Method Staff sử dụng **ID Token** và để trống **Authorization Scopes**. Nếu cấu hình scopes bắt buộc, API Gateway sẽ yêu cầu access token có scopes tương ứng — không đúng với cách frontend hiện tại đang sử dụng.

## 5. Quy ước dữ liệu và lỗi

- **Request/Response:** JSON, `Content-Type: application/json`.
- **Mã trạng thái:** `200` thành công khi đọc/cập nhật, `201` khi tạo, `400` dữ liệu sai, `401` chưa xác thực/token lỗi, `403` không đủ quyền, `409` xung đột trạng thái, `500` lỗi phía server.
- **Đơn hàng:** Dùng `Idempotency-Key` cho `POST /guest/orders` để tránh tạo trùng khi khách bấm lại do lỗi mạng.
- **DynamoDB:** Các bước ảnh hưởng nhiều bảng sử dụng transaction/conditional write nhằm tránh trừ kho hoặc cộng tiền hai lần.
- **Bảo mật:** Không tin `role`, `bill_id` hoặc `table_id` tự khai báo trong các request cần kiểm tra quyền; không ghi token vào CloudWatch Logs.