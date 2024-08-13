---
title: "Xây dựng hệ thống Performance Testing Locust bằng Amazon Elastic Container Service - Phân 1"
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

- AWS Elastic Container Service: Dịch vụ điều phối container được quản lý trên AWS giúp bạn triển khai, quản lý và mở rộng quy mô ứng dụng trong bộ chứa hiệu quả hơn.
- AWS Fargate: Dịch vụ serverless trên AWS chịu trách nhiệm quản lý máy chủ, có thể kích hợp với các dịch vụ khác giúp dễ dàng hơn trong công tác triển khai và quản lý tài nguyên máy chủ.
- AWS CloudWatch: Dịch vụ AWS cho phép bạn giám sát ứng dụng, phản hồi những thay đổi về hiệu suất, tối ưu hóa việc sử dụng tài nguyên và cung cấp thông tin chi tiết về tình trạng hoạt động.
- Cluster Locust: Giải pháp kiểm thử hiệu năng Locust sẽ được triển khai theo mô hình cluster, cho phép giả lập lượng lớn người dùng trong cuộc kiểm thử.
- Ứng dụng đích: Ứng dụng mà chúng ta sẽ thực muốn biết về hiệu năng, hoạt động dựa trên các cuộc kiểm thử.

{{< image src="images/ws_01_high_level.png" caption="Ảnh 1: Kiến trúc triển khai trên AWS" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Hành trình của kiểm thử bắt đầu từ hình thành lên kịch bản kiểm thử. Tiếp theo là bước biến ý tưởng thành hiện thực, định nghĩa chúng trong mã nguồn python. Cuộc kiểm thử sẽ được tiến hành trên môi trường cluster Locust được hình thành trên các dịch vụ của AWS. Chi tiết các bước để tiến hành kiểm thử tự động được đề cập trong bài viết được mô tả dưới đây:

1. Kiểm thử viên viết mã python
2. Nhà vận hành tạo cơ sở hạ tầng cho cluster Locust
3. Nhà vận hành cập nhật kịch bản kiểm thử Locust về EFS
4. Tạo ECS Service
5. Truy cập giao diện Locust, tiến hành chạy Load Testing.

## Yêu cầu chuẩn bị

Bạn sẽ cần chuẩn bị theo các yêu cầu dưới đây để sẵn sàng đi tới chi tiết các bước thực hiện:

- [Tài khoản AWS](https://aws.amazon.com/)
- [Cài đặt công cụ dòng lệnh AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Thiết lập cấu hình bí mật cho AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [Cài đặt Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

## Hướng dẫn chi tiết

Trước khi bắt đầu các bước trong phần chi tiết, bạn thực sự sẽ cần tạo [Key Pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html) chuẩn bị sẵn sàng cho kết nối tới EC2.

### Triển khai ECS Cluster

Để thuận tiện nhất trong bài viết, chúng ta sẽ triển khai hạ tầng ECS Cluster thông qua Terraform. Đồng thời, giữ bài viết ở mức đơn giản, chúng ta sẽ sử dụng backend mặc định cho Terraform, tệp tin state trên máy cá nhân. Mã nguồn Terraform đã sẵn sàng, chúng ta sẽ clone kho lưu trữ này và cung cấp hạ tầng bằng Terraform:

```bash
git clone https://github.com/orez-fu/cloud_resources.git
cd cloud_resources/FCJ/workshop-1/ecs_infra

terraform init
terraform apply -auto-approve
```

Quá trình hình thành cơ sở hạ tầng ECS Cluster và các thành phần đi kèm diễn ra trong khoảng 3 phút. Sau đó bạn sẽ nhận được kết quả về thông tin cho tài nguyên đã tạo:

- Thông tin về bastion gồm địa chỉ IP public và username
- ID của ECS cluster
- ID của EFS

Hãy truy cập AWS Console và kiểm tra các tài nguyên mà Terraform đã tạo ra.

### Cập nhật kịch bản kiểm thử trong EFS

Tại bước này, chúng ta sẽ tải xuống và lưu trữ mã nguồn định nghĩa các kịch bản xuống EFS tạo ở bước trước. Mã nguồn locust cũng là đơn giản để minh họa cho các kịch bản kiểm thử diễn ra trên Locust [sample_locust](https://github.com/orez-fu/sample_locust).

Trong bastion đã thực hiện mount EFS tới đường dẫn `/mnt/efs`. Mã nguồn locust sẽ được tải xuống tới đường dẫn này, và được chia sẻ tới các node trong Locust Cluster ở phần sau. Bạn cần truy cập vào bastion và thực hiện theo các câu lệnh sau để chuẩn bị cũng như là cập nhật kịch bản kiểm thử khi cần:

```bash
sudo su

cd /mnt/efs

git clone https://github.com/orez-fu/sample_locust.git
```

### Tạo Locust Cluster bằng ECS Service

Khi môi trường ECS và kịch bản kiểm thử đã được chuẩn bị, bạn có thể tạo ra Locust Cluster và thực hiện kiểm thử bất cứ khi nào muốn. Quay trở lại mã nguồn Terraform **cloud_resources**, bạn cần đi tới thư mục `FCJ/workshop-1/ecs_service`, nơi chứa các cấu hình tạo nên 2 service trong ECS cluster:

- Service Locust Master: cung cấp giao diện quản trị, điều khiển kịch bản kiểm thử hiệu năng và điều phối giả lập yêu cầu tới các node worker. Service dành cho Locust Master chỉ cần 1 container duy nhất.
- Service Locust Worker: tiếp nhận, thực thi các kịch bản kiểm thử được yêu cầu từ node master. Theo quy mô của cuộc kiểm thử, các container Locust Worker có thể được cố định số lượng trước hoặc tự động co dãn theo tình trạng thực tế lúc kiểm thử thực thi.

Chúng ta sử dụng Terraform tạo Locust Cluster:

```bash
# trên đường dẫn FCJ/workshop-1/ecs_service
terraform init
terraform apply -auto-approve
```

Sẽ mất khoảng 6-10 phút để hoàn tất các tài nguyên cho tiến hành chạy kiểm thử. Kết quả sẽ được hiển thị mang thông tin Domain Name truy cập tới trang web của Locust. Ví dụ về kết quả thu được:

```text
Apply complete! Resources: 19 added, 0 changed, 0 destroyed.

Outputs:

alb_dns_name = "fcj-workshop-1-alb-1283279575.us-east-1.elb.amazonaws.com"
aws_service_master_name = "fcj-workshop-1-ecs-service-master"
```

### Tiến hành chạy Load Testing

Truy cập trang web Locust theo kết quả của bước trước `alb_dns_name`. Bạn sẽ thấy giao diện cơ bản của Locust:

{{< image src="images/3_locust_exp_1.png" caption="Ảnh 2: Trang chủ Locust" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Chúng ta đã triển khai Locust Cluster với 1 master và 2 worker, master đóng vai trò điều phối và hiển thị thông tin, bạn có thể thấy phần thông tin **Workers** với số lượng Locust worker là 2.

Để bắt đầu chạy kiểm thử hiệu năng, chúng ta sẽ điền các giá trị:

- Number of users (peak concurrency): Số lượng user giả lập đồng thời tối đa. Giá trị **50**.
- Ramp up (users started/second): Số lượng user giả lập tăng theo thời gian. Giá trị **5** user giả lập mỗi 1 giây.
- Host: Ứng dụng chúng ta muốn kiểm thử. Giá trị mặc định được khởi tạo trong mã nguồn.

Sau khi nhập vào các giá trị đầu vào của cuộc kiểm thử, ấn nút **Start** để bắt đầu quá trình giả lập yêu cầu người dùng tới hệ thống. Tiếp theo là công việc của các container trong ECS, cụ thể là các container có vai trò Locust worker chạy thuộc service **fcj-workshop-1-ecs-service-worker** trong ECS. Để theo dõi cuộc kiểm thử, chúng ta sẽ có các tab trên giao diện Locust về thống kê, biểu đồ, lỗi, log,...

{{< image src="images/3_locust_exp_2.png" caption="Ảnh 3: Bảng thông số hoạt động" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title 3"  webp="false" >}}

{{< image src="images/3_locust_exp_3.png" caption="Ảnh 4: Đồ thị kiểm thử" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title 4"  webp="false" >}}

Kịch bản kiểm thử hiệu năng của chúng ta thực sự rất đơn giản, 1 API tới trang web tin tức Dân Trí. Một số thông tin nổi bật bạn có thể dễ thấy, RPS và Failures. Trong đó, tỉ lệ lỗi gọi API rất cao, bởi hệ thống của Dân Trí đã có lớp bảo vệ đặt giới hạn các yêu cầu gửi tới hệ thống(rate limiting), đây cũng là cách phổ biến để bảo vệ hệ thống khỏi các cuộc tấn công DDoS. Nếu bạn thực sự muốn kiểm tra khả năng chịu tải của hệ thống, trong khoảng thời gian thực hiện các cuộc kiểm thử hiệu năng, bạn cần phải tạm thời tắt giới hạn truy cập, điều này cho phép các yêu cầu không nhận về mã lỗi HTTP 429(too many requests).

Để có thể ngừng cuộc kiểm thử, bạn ấn nút **Stop**. Các báo cáo thống kê về cuộc kiểm thử cũng luôn sẵn sàng cho bạn tải xuống. Bạn cần lưu ý rằng nút **Reset** sẽ xóa bỏ dữ liệu về cuộc kiểm thử trước đó.

## Dọn dẹp môi trường

Để tránh tổn thất chi phí thêm trong tương lai, chạy các câu lệnh dưới đây để dọn dẹp tài nguyên trên môi trường AWS, sẽ mất khoảng 10-15 phút.

```bash
# Đứng từ thư mục mã nguồn cloud_resouces
cd FCJ/workshop-1/ecs_infra
terraform destroy -auto-approve

cd ../ecs_service
terraform destroy -auto-approve
```

## Tổng kết

Thông qua bài viết này, bạn đã nắm được cách hoạt động của Locust để tiến hành cuộc kiểm thử hiệu năng. Mục đích của kiểm thử hiệu năng nhằm nâng cao và ổn định chất lượng của hệ thống, thông qua kết quả trực quan thu được, đây là nguồn dữ liệu để chúng ta đưa ra các phương hướng cải tiến tiếp theo nếu cần. Tính đơn giản, dễ dùng, khả năng mở rộng tốt là những điểm đáng chú ý của giải pháp Locust.

Hơn nữa, bạn cũng đã tiếp cận với dịch vụ trên AWS như ECS, Fargate, EFS. Hướng tiếp cận này là đơn giản hơn so với hình thành EKS cluster, nhưng vẫn có đủ sức mạnh để cung cấp tài nguyên cho Locust Cluster. AWS ECS dễ dàng vận hành và phù hợp với các ứng dụng, hệ thống đơn giản. Khi kết hợp với Fargate, theo mô hình pay-as-you-go, chúng giúp bạn tiết kiệm chi phí hơn. Trong tình huống của bài viết này đưa ra, chỉ khi cần chạy kiểm thử, chúng ta mới cần các tài nguyên do Fargate tạo ra.

Ở phần tiếp theo, chúng ta sẽ cùng nhau bóc tách cấu trúc của ECS để hiểu hơn về các thành phần trong ECS. Và đồng thời, chúng ta cũng sẽ nâng cấp giải pháp, thêm yếu tố tự động mở rộng, co giãn theo yêu cầu của các cuộc kiểm thử hiệu năng.

Mong rằng bài viết đã cung cấp tới bạn những thông tin bổ ích và ở những bước tiếp sau bài viết này, bạn có thể xây dựng hệ thống kiểm thử và luôn luôn đảm bảo tính tin cậy, độ ổn định trong ứng dụng, hệ thống của bạn.
