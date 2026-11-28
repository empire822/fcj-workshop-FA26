---
title : "Chuẩn bị"
date : 2024-01-01 
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

SecureTable được xây dựng và triển khai bằng tài khoản **AWS Free Plan**, với thời hạn sử dụng tối đa 6 tháng. Tuy nhiên, một số dịch vụ AWS có thể bị giới hạn hoặc yêu cầu xác minh tài khoản trước khi sử dụng.

Trong quá trình triển khai SecureTable, chúng ta có thể gặp phải vấn đề này khi tạo **Amazon CloudFront Distribution**.

## 1. Kiểm tra quyền sử dụng CloudFront

Trước khi bắt đầu Workshop, hãy kiểm tra xem tài khoản AWS có thể tạo CloudFront Distribution hay không.

1. Đăng nhập vào [AWS Management Console](https://console.aws.amazon.com/).
2. Tìm kiếm và truy cập dịch vụ **CloudFront**.
3. Chọn **Create distribution** và tiến hành các bước cấu hình.
4. Kiểm tra xem AWS có cho phép tạo Distribution hay không.

Nếu tài khoản chưa được xác minh, AWS có thể hiển thị thông báo lỗi:

> Your account must be verified before you can add new CloudFront resources. To verify your account, please contact AWS Support (https://console.aws.amazon.com/support/home#/) and include this error message.

Nếu gặp thông báo này, chúng ta cần liên hệ AWS Support để yêu cầu xác minh tài khoản.

## 2. Yêu cầu xác minh tài khoản qua AWS Support

Để yêu cầu mở quyền sử dụng CloudFront, thực hiện các bước sau:

1. Truy cập [AWS Support Center](https://console.aws.amazon.com/support/home#/).
2. Chọn **Create case**.
3. Chọn loại yêu cầu **Account and billing**.
4. Chọn **Account Activation → Account Verification** nếu các tùy chọn này xuất hiện.
5. Nhập tiêu đề, ví dụ: `CloudFront Account Verification Request`.
6. Mô tả vấn đề và đính kèm nguyên văn thông báo lỗi CloudFront.
7. Chọn **Submit** để gửi yêu cầu hỗ trợ.

Sau khi tiếp nhận yêu cầu, AWS Support sẽ kiểm tra tài khoản và xem xét mở quyền sử dụng CloudFront.

Theo kinh nghiệm triển khai, AWS có thể phản hồi trong khoảng **1 ngày làm việc** và hoàn tất xác minh trong khoảng **1–2 ngày làm việc**. Thời gian thực tế có thể thay đổi tùy từng tài khoản.

Khi AWS Support xác nhận đã gỡ bỏ giới hạn, hãy quay lại CloudFront và thử tạo Distribution một lần nữa.

**Lưu ý:**

- Chúng tôi không chắc tất cả tài khoản AWS Free Plan đều gặp giới hạn này.
- Thời gian xử lý phụ thuộc vào quá trình xác minh của AWS và không được bảo đảm.
- Nếu tài khoản đã có quyền tạo CloudFront Distribution, có thể bỏ qua bước này và tiếp tục Workshop.
