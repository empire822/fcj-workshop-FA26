---
title : "Kiểm thử Hệ thống từ đầu đến cuối"
date : 2024-01-01
weight : 9
chapter : false
pre : " <b> 5.9. </b> "
---

Sau khi hoàn thành các thành phần chính của SecureTable, chúng ta sẽ kiểm thử một số luồng nghiệp vụ để xác nhận **CloudFront, Cognito, API Gateway, Lambda và DynamoDB** hoạt động cùng nhau. Ngoài ra, chúng ta sẽ kiểm tra chức năng tự động hóa và khả năng quan sát hệ thống.

## 1. Kiểm thử Frontend và Authentication

1. Truy cập `https://<cloudfront-domain>/` để kiểm tra các trang web được phân phối qua **CloudFront** từ **S3**.
2. Mở trang đăng nhập Staff, đăng nhập bằng tài khoản thuộc nhóm `RECEPTION` hoặc `KITCHEN` của **Amazon Cognito**. Xác nhận hệ thống chuyển đến giao diện tương ứng.
3. Thử gọi một Staff API khi không có token hợp lệ; request phải bị từ chối. Tiếp tục thử truy cập chức năng ngoài quyền của tài khoản đã đăng nhập để kiểm tra phân quyền trong Lambda.

**Kết quả:** Website được tải qua HTTPS; Staff đăng nhập được; API Gateway xác thực token và Lambda kiểm tra `cognito:groups` trước khi cho phép thực hiện nghiệp vụ.

## 2. Kiểm thử luồng đặt món: Customer → Kitchen → Billing

1. Trên giao diện **Reception**, chọn bàn `T01` và mở bill. Trước khi mở bill, thử quét QR của T01 để xác nhận Customer không thể đặt món.
2. Quét QR khi bill đã mở, chọn món và bấm **Gửi bếp**. Hệ thống gọi `POST /v1/guest/orders` thông qua **CloudFront → API Gateway → Ordering Lambda**.
3. Mở **DynamoDB**, kiểm tra các bản ghi mới trong `SecureTable-Orders` và `SecureTable-OrderItems`, cùng trạng thái nguyên liệu được giữ trước.
4. Chuyển sang giao diện **Kitchen**, chọn **Complete** hoặc **Reject** đối với món vừa đặt. Quay lại trang Customer, mở **Món đã đặt** để kiểm tra trạng thái mới.
5. Trên giao diện **Reception**, đóng bill và thử đặt món lại bằng Guest Session cũ. Hệ thống phải từ chối yêu cầu với bill đã đóng.

**Kết quả:** Đơn hàng đi từ Customer đến Kitchen, trạng thái món được cập nhật trong DynamoDB, lượng nguyên liệu được xử lý theo kết quả hoàn tất/từ chối và bill không nhận thêm đơn sau khi đóng.

## 3. Kiểm thử Inventory, Automation và Monitoring

1. Đăng nhập **Inventory**, tạo một phiếu nhập nguyên liệu. Kiểm tra tồn kho thay đổi và xác nhận có bản ghi lịch sử liên quan trong **Audit Dashboard** nếu thao tác đó được ghi audit.
2. Mở **Lambda `SecureTable-Report`**, chờ đến đúng giờ đã đặt trước trong EventBridge, email báo cáo đã được gửi vào tài khoản admin thành công
3. Mở **Lambda `SecureTable-Backup`**, chạy test `dry_run` để kiểm tra điều kiện sao lưu 13 bảng DynamoDB. Với một export đã khởi tạo trước đó, vào **DynamoDB → Exports to S3** để xác nhận trạng thái `COMPLETED`, rồi kiểm tra dữ liệu trong **S3**.
4. Mở **CloudWatch Logs** của Lambda vừa chạy và **CloudTrail Event history** để kiểm tra thông tin thực thi và hoạt động quản trị AWS tương ứng.

Sau khi hoàn thành, chúng ta có thể xác nhận các luồng chính của **SecureTable** đã được kết nối và kiểm thử qua giao diện người dùng, hệ thống API, cơ sở dữ liệu và các dịch vụ tự động hóa.
