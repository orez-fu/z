---
title: "Xây dựng hệ thống Performance Testing Locust bằng Amazon Elastic Container Service"
meta_title: ""
description: ""
date: 2024-07-20T05:00:00Z
image: ""
categories: ["Cloud", "AWS", "Automation Testing"]
author: "Orez Fu"
tags: ["cloud", "aws", "automation", "testing"]
draft: false
---

## Lời mở đầu

Một hệ thống, ứng dụng hoặc trang web phát triển đầy đủ các tính năng vẫn chưa phải phải là một **production hoàn thiện**. Để trở thành một **production**, hệ thống, ứng dụng cần phải đáp ứng được đầy đủ các tiêu chí về các yêu cầu phi chức năng theo tài liệu SRS(System Requirement Specification). Trong bài viết này, chúng ta sẽ đề cập tới yêu cầu phi chức năng về tính ổn định và độ tin cậy của hệ thống, ứng dụng thông qua việc đảm bảo hiệu năng, khả năng chịu tải của chúng. Điều này đặc biệt quan trọng khi một hệ thống phải đối mặt với một số lượng lớn người và có thể cùng lúc sử dụng các tính năng được cung cấp.

Để đánh giá hiệu năng của một hệ thống, ứng dụng, **Performance Testing** là phương pháp phổ biến được sử dụng. Trong performance testing có những loại phục vụ các chủ đích khác nhau, bạn sẽ tìm hiểu tổng quan về chúng trong phần đầu của bài viết. Về mặt công nghệ đáp ứng kỹ thuật này, chúng ta sẽ tiếp cận với giải pháp Locust để tiến hành các cuộc thử nghiệm về hiệu năng. Để có giả lập lượng lớn người dùng truy cập hệ thống, Locust sẽ được triển khai theo dạng cluster với nhiều worker node.

Thông qua bài viết này, chúng tôi sẽ cung cấp các bước chi tiết về tạo cluster Locust trên công nghệ AWS Elastic Container Service. Chúng ta sẽ tập trung đặc biệt vào khả năng của AWS ECS để có thể dễ dàng tạo môi trường và thực hiện lượng lớn các yêu cầu giả lập tới hệ thống, ứng dụng. Sau đó, bạn sẽ sử dụng giao diện web của Locust để yêu cầu tải và trực tiếp xem báo cáo, kết quả.

## Tổng quan về Performance Testing

### Về Performance Testing

Performance Testing là phương pháp kiểm tra, đánh giá hệ thống về khả năng đáp ứng và độ ổn định trong điều kiện workload cụ thể. Thông thường, các cuộc kiểm thử hiệu năng được tiến hành để đánh giá về các thông số tốc độ, độ bền, độ tin cậy và quy mô phục vụ của ứng dụng. Kết quả của Performance Testing sẽ giúp người sử dụng đưa ra được những điều chỉnh hợp lí để cải thiện hiệu năng của hệ thống.

Các ứng dụng khác nhau có thể yêu cầu các thử nghiệm và số liệu khác nhau. Điều quan trọng là cần hiểu loại kiểm thử nào sẽ cung cấp kết quả phù hợp nhất. Chúng tôi sẽ cung cấp tới bạn một vài loại hình kiểm thử phổ biến:

- **Load Testing**: Load Testing được thực hiện để kiểm tra tình trạng hoạt động của ứng dụng khi nó phải nhận các yêu cầu từ một số lượng người dùng cố định. Ví dụ: bạn thử nghiệm với 1000 người dùng đồng thời sử dụng dịch vụ web trong một khoảng thời gian hữu hạn.
- **Stress Testing**: Stress testing khá giống với load testing, ngoại trừ việc nó rõ ràng nhắm tới tìm ra mức tối đa về mặt hiệu năng mà hệ thống có thể đạt tới được. Stress testing cũng là cách thức để biết về phản ứng của hệ thống trong tình trạng quá tải, liệu hệ thống phản ứng ra sao. Nói tóm lại, bạn thử nghiệm với số lượng yêu cầu người dùng ngày càng tăng cho đến khi nó tiệm cận với sự cố quá tải.
- **Endurance Testing**: Endurance Testing hay còn gọi là Soak Testing là phương pháp kiểm thử ứng dụng trong điều kiện thực tế hoặc tương tự như ở môi trường người dùng cuối trong một khoảng thời gian dài. Ví dụ: bạn tiến hành cuộc kiểm thử với số lượng người dùng cho các tính năng chính của ứng dụng trong một khoảng thời gian dài, tối thiểu 24 giờ.

## Về giải pháp Locust

Trích dẫn trực tiếp từ trang chủ của Locust, "Locust is an easy to use, scriptable and scalable performance testing tool". Ý tưởng của công cụ này là dùng một nhóm các máy cài đặt Locust để có thể giả lập nhiều các yêu cầu tới hệ thống. Các hành vi giả lập yêu cầu của người dùng được định nghĩa bằng các kịch bản viết bằng mã nguồn python. Locust cung cấp giao diện web cho phép cấu hình, theo dõi quá trình thực hiện kiểm thử theo thời gian thực, và các kết quả cũng được trực quan hóa trên đó.

Dưới đây là các tính năng chính của Locust:

- **Python-Based**: Các kịch bản người dùng được viết bằng Python, linh hoạt khi tạo các kịch bản thực tế, phức tạp.
- **Distributed & Scalable**: Locust có thể giả lập hàng triệu người dùng đồng thời bằng cách phân phối tải tới nhiều máy thực thi.
- **Web-Based UI**: Locust cung cấp một giao diện người dùng đơn giản, cho phép bạn bắt đầu, dừng, và giám sát các cuộc kiểm thử theo thời gian thực.
- **Real-Time Reporting**: Giao diện web thể hiện kết quả kiểm thử theo thời gian thưc, bao gồm, thời gian phản hồi, số lượng người dùng, tốc độ tăng của yêu cầu và tỉ lệ lỗi xuất hiện.
- **Support for Any System**: Locust không giới hạn về giao thức sử dụng khi tạo các yêu cầu tới hệ thống, bạn có thể kiểm thử tới bất kỳ hệ thống hay giao thức nào (HTTP, HTTPS, WebSocket, ...) bằng cách viết mã phù hợp.
- **CLI & Headless Mode**: Hỗ trợ CLI với đa dạng các tùy biến cho phép bạn có thể thực thi kiểm hoàn toàn bằng giao diện dòng lệnh. Điều này cho phép bạn dễ dàng tích tự động hóa và tích hợp với các CI/CD Pipeline.
- **Detailed Statistics**: Locust thu thập số liệu thống kê chi tiết về từng loại yêu cầu, cung cấp thông tin sát sao về điểm bottleneck ảnh hưởng tới hiệu năng hệ thống.
- **Scheduling & User Load Patterns**: Các tùy chọn lập lịch linh hoạt để tăng hoặc giảm số lượng người dùng theo các kế hoạch được xác định trước.

## Tổng quan giải pháp

Giải pháp được đề xuất trong bài viết sử dụng một số công cụ AWS để thiết lập môi trường kiểm thử hiệu năng nhằm đáp ứng khả năng mở rộng tốt, triển khai đơn giản và duy trì ở mức chi phí thấp:

- AWS Elastic Container Service
- AWS Fargate
- AWS CloudWatch
- Cluster Locust
- Ứng dụng đích

Hành trình của kiểm thử bắt đầu từ hình thành lên kịch bản kiểm thử. Tiếp theo là bước biến ý tưởng thành hiện thực, định nghĩa chúng trong mã nguồn python. Cuộc kiểm thử sẽ được tiến hành trên môi trường cluster Locust được hình thành trên các dịch vụ của AWS. Chi tiết các bước để tiến hành kiểm thử tự động được đề cập trong bài viết được mô tả dưới đây:

1. Kiểm thử viên viết mã python
2. Nhà vận hành tạo cơ sở hạ tầng cho cluster Locust
3. Nhà vận hành cập nhật kịch bản kiểm thử Locust về EFS
4. Tạo ECS Service
5. Truy cập giao diện Locust, tiến hành chạy Load Testing.

## Yêu cầu chuẩn bị

Bạn sẽ cần chuẩn bị theo các yêu cầu dưới đây để sẵn sàng đi tới chi tiết các bước thực hiện:

- Tài khoản AWS
- Cài đặt công cụ dòng lệnh AWS CLI và thiết lập cấu hình cho AWS CLI
- Cài đặt Terraform

## Hướng dẫn chi tiết

### Triển khai ECS Cluster

### Cập nhật kịch bản kiểm thử trong EFS

### Tạo Locust Cluster bằng ECS Service

### Tiến hành chạy Load Testing

## Dọn dẹp môi trường

## Tổng kết
