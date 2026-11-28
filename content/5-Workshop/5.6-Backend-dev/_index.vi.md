---
title : "Xây dựng Backend"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

Hệ thống sử dụng kiến trúc **Serverless**: không cần quản lý EC2 hay web server chạy liên tục. Các Lambda được gọi khi có request API hoặc sự kiện theo lịch.

## 1. Kiến trúc xử lý Backend

Có hai nhóm luồng chính:

1. **Guest:** Customer quét QR → CloudFront → API Gateway → Lambda GuestServices/Ordering → DynamoDB. Guest không đăng nhập Cognito; Lambda kiểm tra **Guest Session Token** và trạng thái bill.
2. **Staff:** Nhân viên đăng nhập bằng Cognito → CloudFront → API Gateway (**Cognito Authorizer**) → Lambda tương ứng → DynamoDB. Ngoài xác thực, Lambda còn kiểm tra nhóm `RECEPTION`, `KITCHEN`, `INVENTORY` hoặc `ADMIN`.

Các request từ frontend sử dụng đường dẫn `/v1/*`. Trong CloudFront, đường dẫn này được chuyển tới **API Gateway REST API** thay vì S3.

## 2. Các Lambda của SecureTable

Sau khi gộp các function có cùng nghiệp vụ, SecureTable tổ chức backend thành **8 Lambda**:

| Lambda | Nhiệm vụ chính | Cách kích hoạt |
|---|---|---|
| `SecureTable-GuestServices` | Cấp Guest Session, trả thực đơn | API Gateway – Guest |
| `SecureTable-Ordering` | Nhận đơn, giữ nguyên liệu, xem lại món đã đặt | API Gateway – Guest |
| `SecureTable-Reception` | Xem bàn, mở/đóng bill, xem hóa đơn | API Gateway – Staff |
| `SecureTable-Kitchen` | Xem hàng đợi, hoàn tất/từ chối món | API Gateway – Staff |
| `SecureTable-Inventory` | Nhập kho, xem phiếu nhập, điều chỉnh tồn | API Gateway – Staff |
| `SecureTable-AdminDashboard` | Tổng hợp tài chính và truy vấn Audit Logs | API Gateway – Staff |
| `SecureTable-Report` | Tạo và gửi báo cáo định kỳ | EventBridge Scheduler |
| `SecureTable-Backup` | Khởi tạo sao lưu DynamoDB sang S3 | EventBridge Scheduler |

**Lưu ý:** `Report` và `Backup` không cần API công khai. Việc tạo lịch EventBridge, cấu hình SNS và S3 backup được thực hiện ở [5.7 – Tự động hóa Báo cáo & Sao lưu dữ liệu](../5.7-Automation).

## 3. Thứ tự triển khai

Chúng ta sẽ thực hiện theo ba phần:

1. [5.6.1 – Tổng quan Backend & Thiết kế API](../5.6-Backend-dev/5.6.1-Overview): Xác định những endpoint mà Guest và Staff cần, phương thức HTTP và Lambda xử lý.
2. [5.6.2 – Tạo và Cấu hình các AWS Lambda Functions](../5.6-Backend-dev/5.6.2-Lambda): Tạo function, upload mã nguồn, cấu hình runtime và cấp IAM Execution Role.
3. [5.6.3 – Tích hợp Lambda với Amazon API Gateway](../5.6-Backend-dev/5.6.3-APIgw): Tạo REST API, gắn Lambda proxy integration, bật Cognito Authorizer và cấu hình CloudFront để frontend gọi API.

**Lưu ý:** Các bảng DynamoDB phải được tạo đúng tên và khóa tại mục 5.4 trước khi kiểm thử Lambda. Ngoài ra, frontend sẽ chỉ gọi được API khi API Gateway đã được deploy và behavior `/v1/*` của CloudFront được cấu hình đúng.