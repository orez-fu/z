---
title: "Đơn giản hóa triển khai Amazon EKS với GitHub Actions và AWS CodeBuild"
meta_title: ""
description: "this is meta description"
date: 2022-04-04T05:00:00Z
image: ""
categories: ["Cloud", "AWS"]
author: "Orez Fu"
tags: ["vi", "cloud", "aws"]
draft: false
---

> Đây là bản dịch cho bài viết [Simplify Amazon EKS Deployments with GitHub Actions and AWS CodeBuild](https://aws.amazon.com/blogs/devops/simplify-amazon-eks-deployments-with-github-actions-and-aws-codebuild/).

Trong bối cảnh kỹ thuật, công nghệ thông tin phát triển nhanh chóng ngày nay, các tổ chức chuyển mình bằng cách áp dụng [các kỹ thuật trong DevOps](https://aws.amazon.com/devops/what-is-devops/) để thúc đẩy đổi mới và làm mượt quy trình phát triển phần mềm và quản lý cơ sở hạ tầng. Một yếu tố then chốt trong DevOps là Continuous Integration và Continuous Delivery (CI/CD - tích hợp, triển khai liên tục), cho phép tự động hóa các hoạt động trong triển khai để giảm thiểu thời gian cho việc phát hành các bản cập nhật ứng dụng. AWS đề xuất một bộ các công cụ để hỗ trợ CI/CD, có đầy đủ tính năng, sự linh hoạt và khả năng tùy biến thông qua tích hợp với các công cụ bên thứ ba.

Thông qua bài viết này, bạn sẽ học cách sử dụng GitHub Actions để tạo CI/CD workflow với AWS CodeBuild và AWS CodePipeline. Bạn sẽ tận dụng khả năng của GitHub Actions đến từ vô số các actions có sẵn trong GitHub Marketplace để xây dựng và triển khai một ứng dụng Python tới cụm Amazon Elastic Kubernetes Service (EKS).

GitHub Actions là một tính năng mạnh mẽ trên nền tảng phát triển của GitHub, nó cho phép bạn tự động hóa các workflow liên quan đến phát triển phần mềm trực tiếp trên repository của bạn. Với Actions, bạn có thể viết các tác vụ riêng để xây dựng, kiểm thử, đóng gói, phát hành, triển khai mã nguồn của bạn, và sau đó kết hợp chúng trong workflow tùy biến để làm mượt quá trình phát triển ứng dụng của bạn.


## Tổng quan về giải pháp

Giải pháp này được đề xuất trong bài viết sử dụng một số công cụ AWS dành cho nhà phát triển để thiết lập một CI/CD pipeline trong khi vẫn đảm bảo lộ trình hợp từ phát triển tới triển khai:

- [AWS CodeBuild](https://aws.amazon.com/codebuild/): Dịch vụ xây dựng, đóng gói mã nguồn. Nó có thể biên dịch mã nguồn, chạy kiểm thử, và tạo các bản đóng gói của ứng dụng, cái mà sẽ sẵn sàng cho việc triển khai.
- [AWS CodePipeline](https://aws.amazon.com/codepipeline/): Dịch vụ continuous delivery cho phép điều phối các hoạt động xây dựng, kiểm thử và triển khai trong quy trình triển khai mà bạn cần.
- [Amazon Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/): Dịch vụ được quản trị trên AWS, làm đơn giản việc tạo và sử dụng cụm Kubernetes trên AWS mà không cần cài đặt và vận hành thành phần Kubernetes Control Plane.
- [AWS CloudFormation](https://aws.amazon.com/cloudformation/): AWS CloudFormation cho phép bạn mô hình hóa, cung cấp và quản lý AWS cũng như tài nguyên của bên thứ 3 bằng cách xử lý cơ sở hạ tầng dưới dạng mã. Bạn sẽ sử dụng dịch vụ này để triển khai các tài nguyên cơ bản được yêu cầu trong phần chuẩn bị.
- [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/): Dịch vụ Container Registry trên AWS, tương tự như Docker Hub, tạo điều kiện thuận tiện cho nhà phát triển để lưu trữ, quản lý, và triển khai Docker container images.

  *Ảnh 1: Kiến trúc workflow thể hiện mã nguồn và các giai đoạn xây dựng, kiểm thử, phê duyệt, triển khai*
[!architecture](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2024/04/29/gha-arch.png)

Hành trình của mã nguồn bắt đầu từ môi trường phát triển - của nhà phát triển - đi đến ứng dụng - hướng tới người dùng - là một quá trình chuyển tiếp liên tục trên các dịch vụ AWS khác nhau với các hoạt động chính gồm xây dựng và triển khai được thực hiện thông qua GitHub Actions:

1. Nhà phát triển
2. Hệ thống quản lý mã nguồn (SCM) 
3. AWS CodePipeline
4. Khi container image 
5. Giai đoạn phê duyệt
6. Sau khi nhận được thông tin phê duyệt
7. Trong giai đoạn triển khai,
8. Ứng dụng được triển khai

## Bước chuẩn bị

## Hướng dẫn chi tiết

### Triển khai các thành phần cơ bản

### Clone mã nguồn từ CodeCommit Repository

### Phát triển ứng dụng

## Tổng quan về CI/CD Pipeline

### Định nghĩa GitHub Actions trong AWS CodeBuild

### Đặc tả trong AWS CodeBuild - Giai đoạn đóng gói/xây dựng

### Đặc tả trong AWS CodeBuild - Giai đoạn triển khai

### Nghiệm thu ứng dụng

## Dọn dẹp môi trường

## Tổng kết

## Tác giả

