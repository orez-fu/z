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

annotation
backend cho Terraform
manifest YAML
deployment YAML
docker image
pod
canary



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

1. [Một tài khoản AWS](https://aws.amazon.com/)
2. [Cài đặt giao diện dòng lệnh AWS (AWS CLI) version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
3. [Thiết lập cấu hình bí mật cho AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
4. [Cài đặt Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
5. [Cài đặt Kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
6. [Cài đặt Docker](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-docker.html)

## Hướng dẫn chi tiết

### Kích hoạt Application Signals

Làm theo hướng dẫn để [kích hoạt Application Signal](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Signals-Enable-EC2.html#CloudWatch-Application-Signals-EC2-Grant) trong tài khoản AWS của bạn.

### Triển khai ứng dụng bằng Terraform

1. Chúng ta sẽ **cấu hình biến môi trường** được yêu cầu để triển khai ứng dụng bằng Terraform, đồng thời có sử dụng **Amazon S3 bucket** làm backend bằng các câu lệnh dưới đây.

```bash
export AWS_REGION=<your-aws-region>

aws s3 mb s3://tfstate-$(uuidgen | tr A-Z a-z)

export TFSTATE_KEY=application-signals/demo-applications
export TFSTATE_BUCKET=$(aws s3 ls --output text | awk '{print $3}' | grep tfstate-)
export TFSTATE_REGION=$AWS_REGION

export TF_VAR_cluster_name=app-signals-demo
export TF_VAR_cloudwatch_observability_addon_version=v1.5.1-eksbuild.1
```

2. Tiếp theo, chúng ta sẽ **clone kho lưu trữ mã nguồn của ứng dụng** và **triển khai cơ sở hạ tầng bằng Terraform**. Sẽ mất khoảng 15-20 phút để hoàn tất tạo các tài nguyên trên AWS.

```bash
git clone https://github.com/aws-observability/application-signals-demo
cd application-signals-demo/terraform/eks

terraform init -backend-config="bucket=${TFSTATE_BUCKET}" -backend-config="key=${TFSTATE_KEY}" -backend-config="region=${TFSTATE_REGION}"
terraform apply --auto-approve
```

### Cấu hình kubectl

Chạy lệnh dưới đây để cập nhật tệp tin `kubeconfig` nhằm thêm thông tin truy cập Amazon EKS Cluster.

```bash
aws eks update-kubeconfig --name $TF_VAR_cluster_name --region $AWS_REGION --alias $TF_VAR_cluster_name
```

### Triển khai các tài nguyên trên Kubernetes

1. Để bật tính năng Application Signals trên ứng dụng Python, bạn cần thêm annotation `instrumentation.opentelemetry.io/inject-python: 'true'` vào manifest YAML cho ứng dụng trong cụm EKS. Việc thêm annotation này sẽ tự động điều khiển ứng dụng gửi metric, trace và log tới Application Signals.

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      io.kompose.service: billing-service-python
  template:
    metadata:
      labels:
        io.kompose.service: billing-service-python
      annotations:
        instrumentation.opentelemetry.io/inject-python: 'true' 
```

2. Các tệp tin deployment YAML nằm trong thư mục `/demo-app/k8s/`, đã bao gồm các annotation. Triển khai tất cả tài nguyên bằng trên cụm EKS bằng các câu lệnh dưới đây. Trước tiên, sẽ cần biên dịch ứng dụng, đẩy docker image tới kho lưu trữ docker trên ECR và triển khai cụm EKS.

```bash
cd ../..
./mvnw clean install -P buildDocker

export ACCOUNT=`aws sts get-caller-identity | jq .Account -r`
export REGION=$AWS_REGION

./push-ecr.sh
./scripts/eks/appsignals/tf-deploy-k8s-res.sh
```

### Xác nhận kết quả triển khai

1. Chạy câu lệnh dưới đây để **xác nhận triển khai thành công** các ứng dụng. Bạn sẽ thấy danh sách các pod ở trạng thái **Running**.

```bash
kubectl get pods

#Output
NAME                                        READY   STATUS    RESTARTS      AGE
admin-server-java-5c57ddcb46-t4b9l          1/1     Running   0             7m1s
billing-service-python-6bf9766cfc-5g67s     1/1     Running   0             6m52s
config-server-58d94894-dzdhz                1/1     Running   0             6m47s
customers-service-java-69c5d75cc9-5hwrw     1/1     Running   0             6m42s
customers-service-java-69c5d75cc9-tfrts     1/1     Running   0             6m43s
discovery-server-d6bff754f-xgrxv            1/1     Running   0             6m36s
insurance-service-python-6745799b9b-4hdxc   1/1     Running   0             6m33s
pet-clinic-frontend-java-5696d89cd8-cpvj2   1/1     Running   0             6m56s
pet-clinic-frontend-java-5696d89cd8-mlggq   1/1     Running   0             6m56s
vets-service-java-5b6969b8d6-cfvsb          1/1     Running   0             6m29s
visits-service-java-85b9c5c45-p57m5         1/1     Running   0             6m25s
visits-service-java-85b9c5c45-vwkht         1/1     Running   0             6m25s
visits-service-java-85b9c5c45-vx6gj         1/1     Running   0             6m25s
```

2. Tiếp theo chạy lệnh sau để **lấy thông tin URL của ứng dụng**. Mở URL trên trình duyệt web và **khám phá ứng dụng**. Sẽ mất 2-3 phút để URL hoạt động.

```bash
echo "http://$(kubectl get ingress -o json --output jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')"
```

### Tạo canary trong CloudWatch Synthetics để giả lập lưu lượng truy cập

Tiếp theo, chúng ta sẽ **tạo các canary** bằng đoạn lệnh dưới đây, đoạn lệnh sẽ chạy trong khoảng 10 phút để tạo ra lưu lượng truy cập tới ứng dụng, giả lập hành vi truy cập ứng dụng từ phía người dùng ứng dụng.

```bash
endpoint=$(kubectl get ingress -o json  --output jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')

cd scripts/eks/appsignals/
./create-canaries.sh $AWS_REGION create $endpoint
```

## Trực quan hóa ứng dụng bằng CloudWatch Application Signals

Điều hướng tới [bảng điều khiển CloudWatch](https://console.aws.amazon.com/cloudwatch/) và chọn **Services** bên dưới phần **Application Signals** ở menu điều hướng bên trái.

### Service Dashboard

CloudWatch Application Signals tự động khám phá và liên kết các dịch vụ mà không yêu cầu cấu hình thêm trong bảng Services. Góc nhìn này thống nhất, tập trung vào ứng dụng giúp cung cấp cái nhìn toàn cảnh về cách người dùng tương tác với dịch vụ của bạn. Điều này có thể giúp bạn phân loại các vấn đề khi có bất thường về hiệu năng trong hệ thống.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/Services-Console-1.png" caption="Ảnh 3: Service Dashboard" alt="Service Dashboard" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

#### Chi tiết thông tin dịch và các phụ thuộc

Khi chọn một dịch vụ, bạn sẽ chuyển hướng tới trang thông tin chi tiết của dịch vụ có bật Application Signal, hiển thị các thông tin bao gồm: tổng quan, tình trạng vận hành, các thành phần phụ thuộc, các canary và các yêu cầu từ phía người dùng.

Trong **Ảnh 4**, phần Service Overview tổng quát về các thành phần tạo nên dịch vụ của bạn và làm nổi bật các metric để giúp bạn xác định sự cố, là các thông tin trong quá trình khắc phục sự cố.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/Services-Overview-1.png" caption="Ảnh 4: Service Overview" alt="Service Overview" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Điều hướng tới **tab Service operations**, chọn **operation**, và click vào một điểm thời gian trên biểu đồ metric để mở ra phần thông tin về **Correlated trace**, **Top contributor** và **Application Log** liên kết với điểm được chọn.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/ServiceOperations_Latest.png" caption="Ảnh 5: Service operations" alt="Service operations" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Chọn **Trace ID** sẽ điều hướng bạn tới thông tin chi tiết Trace, nơi bạn sẽ tìm thấy bản đồ trace AWS X-ray, hiển thị tất cả các dịch vụ có liên kết với trace ID này.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/trace-correlation.png" caption="Ảnh 6: Trực quan hóa trace tương quan với metric trong vận hành dịch vụ" alt="Service operations" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Phần **Top Contributors** trực tiếp hiển thị các metric cho **Khối lượng yêu cầu (Call Volume)**, **Tính khả dụng (Availability)**, **Độ trễ trung bình (Average Latency)**, **Lỗi (Errors)** và **Sai lầm (Fault)**, được chia nhỏ theo các thành phần trong cơ sở hạ tầng. Tab **Application Logs** cho bạn biết câu truy vấn Log Insight để tìm kiếm và thấy được log liên quan trong ứng dụng.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/04/28/Figure-10-1.png" caption="Ảnh 7: Top Contributors và Application Logs" alt="Service operations" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Với một vài click sẽ hiển thị thông tin liên quan về trace. Điều này cho phép bạn hiểu về các vấn đề gốc rễ mà không cần các câu truy vấn traces riêng rẽ thủ công.

### Service Map

Để hiển thị Service Map, mở [bảng điều khiển CloudWatch](https://console.aws.amazon.com/cloudwatch/) và chọn **Service Map** bên dưới phần **Application Signals** trong khung điều hướng bên trái. Chọn node dịch vụ `billing-service-python` như **Ảnh 8** để hiển thị các kết nối với các dịch vụ và thành phần phụ thuộc, giúp bạn hiểu về mô hình và luồng thực thi trong ứng dụng. Điều này đặc biệt hữu dụng nếu có các dịch vụ không do đội ngũ của bạn phát triển. 

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/Service-Map.png" caption="Ảnh 8: Hiển thị mô hình ứng dụng sử dụng Service Map" alt="Application topology" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

### Service level objectives (SLOs)

Sử dụng Application Signals để định nghĩa Service Level Objectives (SLOs) cho vận hành các nghiệp vụ trọng yếu. Bằng các định nghĩa SLOs cho các dịch vụ, bạn có được khả năng giám sát chúng trên SLO dashboard, cung cấp tổng quan về các hoạt động quan trọng trong vận hành dịch vụ. Các điều kiện SLO bao gồm độ trễ, tính khả dụng và metric trong CloudWatch, cung cấp khả năng theo dõi toàn diện.

Thực hiện theo các [bước tạo SLO](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-ServiceLevelObjectives.html#CloudWatch-ServiceLevelObjectives-Create) để tạo SLOs cho ứng dụng PetClinic.

{{< image src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/02/SLO.png" caption="Ảnh 9: Tạo và trực quan hóa Service level objectives (SLOs)" alt="Application topology" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

## Dọn dẹp môi trường

Chú ý: Các giá trị biến môi trường được xác định trước đó là bắt buộc để có thể xóa ứng dụng thành công.

Để tránh chịu các chi phí thêm, chạy các cấu lệnh dưới đây để dọn dẹp ứng dụng và tài nguyên trên môi trường AWS, sẽ mất khoảng 15-20 phút.

```bash
cd ../../..
./scripts/eks/appsignals/create-canaries.sh $AWS_REGION delete
kubectl delete -f ./scripts/eks/appsignals/sample-app/alb-ingress/petclinic-ingress.yaml

cd ./terraform/eks
terraform destroy --auto-approve
```

## Tổng kết

Trong bài viết này, bạn đã hiểu rõ hơn về cách tận dụng CloudWatch Application Signals để giám sát các ứng dụng trong cụm Amazon EKS một cách liền mạch, có tính liên kết thông tin, mà không yêu cầu chỉnh sửa mã nguồn. Khả năng mạnh mẽ này cho phép bạn dễ dàng thu thập các golden metric (khối lượng yêu cầu, tính khả dụng, độ trễ, lỗi và sai lầm) và dữ liệu trace cho các dịch vụ của bạn, nâng cao khả năng quan sát và tạo điều kiện khắc phục sự cố một cách hiệu quả.

Hơn nữa, chúng ta đã khám phá cách trực quan hóa các hoạt động tổng quan và tình trạng hoạt động của các ứng dụng, dịch vụ bằng các dashboard đã được dựng sẵn bởi Application Signals. Bằng cách tận dụng các dashboard này, bạn có thể dễ dàng truy cập các số liệu về hiệu năng chính và liên kết chúng với dữ liệu trace, cho phép bạn nhanh chóng xác định và giải quyết mọi vấn đề cơ bản chỉ bằng vài click. Ở bước tiếp theo, chúng tôi khuyến khích bạn dùng thử Applicationo Signal với môi trường của bạn.

Vui lòng tham chiếu tới [Tài liệu CloudWatch Application Signals](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html) để khám phá thêm nhiều thông tin hoặc xem [các ca sử dụng CloudWatch Application Signals](https://catalog.workshops.aws/observability/en-US/use-cases/application-signals) trong [Hội thảo về Observability] để có trải nghiệm thực tế.



