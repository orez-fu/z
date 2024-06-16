---
title: "Giám sát ứng dụng Python bằng Amazon CloudWatch Application Signals"
meta_title: ""
description: "this is meta description"
date: 2022-04-04T05:00:00Z
image: ""
categories: ["Cloud", "AWS", "Monitoring"]
author: "Orez Fu"
tags: ["cloud", "aws", "monitoring"]
draft: false
---

> Đây là bản dịch cho bài viết [Monitor Python apps with Amazon CloudWatch Application Signals (Preview)](https://aws.amazon.com/blogs/mt/monitoring-python-apps-using-amazon-cloudwatch-application-signals/)

Bảng thuật ngữ
framework
metrics
trace
dashboard


AWS đã công bố [Amazon CloudWatch Application Signals](https://repost.aws/articles/ARTvHbg1TfRMijt-V0YGSudw/observe-your-applications-with-amazon-cloudwatch-application-signals-preview) trong sự kiện re:Invent 2023. Đây là một tính năng giúp giám sát và hiểu rõ tình trạng của các ứng dụng Java. Ngày hôm nay, chúng tôi vui mừng thông báo rằng hiện tại Application Signals đã hỗ trợ các [ứng dụng Python](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Signals-supportmatrix.html). Việc kích hoạt Application Signals cho phép sử dụng "AWS Distro for OpenTelemetry" (ADOT) để đo lường các ứng dụng Python mà không cần thay đổi mã ứng dụng. Điều này cho phép bạn thu thập các metric và trace chính cho các thư viện và nền tảng được phát triển bằng Python. Điều này cho phép bạn nhanh chóng phân loại tình trạng trong quá trình vận hành và giám sát các mục tiêu về hiệu năng ứng dụng, mà không cần viết thêm các mã tùy chỉnh hay tạo dashboard.

Trong bài viết này, chúng tôi sẽ cung cấp các bước chi tiết về cách tích hợp Application Signals với các ứng dụng Python được triển khai trên một cụm Amazon EKS. Đặc biệt, chúng tôi sẽ tập trung vào việc sử dụng tích hợp này để giám sát các ứng dụng Python được phát triển dựa trên nền tảng [Django](https://www.djangoproject.com/) và tận dụng các thư viện phổ biến như [psycopg2](https://pypi.org/project/psycopg2/), [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) và [requests](https://pypi.org/project/requests/). Sau đó, chúng tôi sẽ trực quan hóa tình trạng ứng dụng bằng bảng điều khiển Application Signals. 

## Tổng quan giải pháp

Dưới đây là tổng quan chi tiết của giải pháp:
- Ứng dụng demo được xây dựng bằng nền tảng Spring Cloud và Django, trong đó, mỗi dịch vụ tự đăng ký với dịch vụ [Eureka discovery-service](https://spring.io/guides/gs/service-registration-and-discovery). Mã nguồn của ứng dụng có thể được tìm thấy trên [GitHub repository](https://github.com/aws-observability/application-signals-demo/).
- Chúng ta có 2 dịch vụ `insurances` và `billing` đều được viết trên nền tảng Django. Các dịch vụ này hiển thị các API thông qua nền tảng Django REST và gọi tới các dịch vụ bên ngoài bằng các thư viện.
- Các dịch vụ cũng có sự tương tác với Amazon RDS PostgreSQL thông qua thư viện psycopg2 và lưu trữ các thông tin thanh toán trong AWS DynamoDB thông qua thư viện boto3.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/image-20.png" caption="Ảnh 1: Các thư viện và nền tảng Python sử dụng trong ứng dụng demo" alt="Thư viện Python sử dụng trong Demo" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Chúng ta sẽ sử dụng Terraform để triển khai các tài nguyên được hiển thị ở **Ảnh 2**. Chúng tôi sẽ sử dụng thêm add-on [Amazon CloudWatch Observability EKS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-addon.html) để triển khai CloudWatch agent và Fluent Bit thông qua tài nguyên DaemonSet để điều phối metric, log, và trace.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/04/28/Figure-2-4.png" caption="Ảnh 2: Kiến trúc giải pháp" alt="Solution Architecture" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

## Yêu cầu chuẩn bị

## Hướng dẫn chi tiết

### Kích hoạt Application Signals

### Triển khai ứng dụng bằng Terraform

### Cấu hình kubectl

### Triển khai các tài nguyên trên Kubernetes

### Xác nhận kết quả triển khai

### Tạo canary trong CloudWatch Synthetics để giả lập lưu lượng truy cập

## Trực quan hóa ứng dụng bằng CloudWatch Application Signals

### Service Dashboard

### Service Map

### Service level objectives (SLOs)

## Dọn dẹp môi trường

## Tổng kết



