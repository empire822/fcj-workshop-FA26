---
title: "Workshop"
date: 2026-11-23
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Xây dựng hệ thống quản lý nhà hàng SecureTable trên nền tảng AWS Serverless

#### Tổng quan

**AWS Serverless** cho phép xây dựng và triển khai ứng dụng mà không cần trực tiếp quản lý máy chủ. Các dịch vụ như AWS Lambda, Amazon API Gateway và Amazon DynamoDB hỗ trợ xử lý yêu cầu, lưu trữ dữ liệu và mở rộng theo nhu cầu sử dụng, giúp giảm công sức vận hành.

Trong workshop này, chúng ta sẽ tìm hiểu cách xây dựng và triển khai **SecureTable**, một hệ thống quản lý nhà hàng trên AWS, tập trung vào quản lý nguyên liệu, gọi món bằng mã QR, xử lý đơn hàng, quản lý hóa đơn và theo dõi doanh thu.

Hệ thống sử dụng kiến trúc Serverless, kết hợp các dịch vụ AWS để xây dựng giao diện web, xử lý nghiệp vụ, xác thực người dùng, lưu trữ dữ liệu và giám sát hoạt động.

Các thành phần chính được triển khai gồm:

- **Frontend – Amazon S3 & Amazon CloudFront:** Lưu trữ và phân phối giao diện web cho khách hàng và nhân viên qua HTTPS.
- **Authentication – Amazon Cognito & AWS IAM:** Xác thực nhân viên, phân quyền và kiểm soát quyền truy cập tài nguyên AWS.
- **Backend – AWS Lambda & Amazon API Gateway:** Xây dựng API phục vụ đặt món, quản lý bàn, hóa đơn, bếp, kho và báo cáo.
- **Database – Amazon DynamoDB:** Lưu trữ dữ liệu nhà hàng và sử dụng DynamoDB Transactions cho các thao tác cần đảm bảo tính nhất quán.
- **Automation – Amazon EventBridge Scheduler & Amazon SNS:** Lập lịch báo cáo, sao lưu dữ liệu và gửi thông báo qua email.
- **Monitoring & Security – Amazon CloudWatch, AWS CloudTrail & AWS WAF:** Theo dõi hệ thống, ghi nhận hoạt động AWS và bổ sung biện pháp bảo vệ web khi WAF được bật.

Sau khi hoàn thành workshop, chúng ta sẽ có một hệ thống SecureTable hoạt động trên AWS: khách hàng quét QR để đặt món, nhân viên thực hiện các nghiệp vụ quản lý và người quản trị theo dõi dữ liệu vận hành.

Workshop cũng trình bày cách kiểm thử, giám sát và tối ưu tài nguyên dựa trên các nguyên tắc của **AWS Well-Architected Framework**.


#### Nội dung

1. [Tổng quan & Kiến trúc hệ thống](5.1-System-overview/)
2. [Chuẩn bị](5.2-Prerequisite/)
3. [Triển khai Frontend](5.3-Frontend-dev/)
4. [Thiết lập cơ sở dữ liệu](5.4-Database-setup/)
5. [Xác thực & Phân quyền](5.5-Auth/)
6. [Xây dựng Backend](5.6-Backend-dev/)
7. [Tự động hóa Báo cáo & Sao lưu dữ liệu](5.7-Automation/)
8. [Tăng cường Bảo mật & Giám sát hệ thống](5.8-Security/)
9. [Kiểm thử Hệ thống từ đầu đến cuối](5.9-E2E-testing/)
10. [Dọn dẹp Tài nguyên](5.10-Cleanup/)
