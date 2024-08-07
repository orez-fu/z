---
title: "Đơn giản hóa triển khai Amazon EKS với GitHub Actions và AWS CodeBuild"
meta_title: ""
description: "this is meta description"
date: 2024-06-26T12:00:00Z
image: ""
categories: ["Cloud", "AWS", "CI/CD"]
author: "Orez Fu"
tags: ["cloud", "aws", "cicd"]
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

{{< image src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2024/04/29/gha-arch.png" caption="Ảnh 1: Kiến trúc workflow thể hiện mã nguồn và các giai đoạn xây dựng, kiểm thử, phê duyệt, triển khai" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Hành trình của mã nguồn bắt đầu từ môi trường phát triển - của nhà phát triển - đi đến ứng dụng - hướng tới người dùng - là một quá trình chuyển tiếp liên tục trên các dịch vụ AWS khác nhau với các hoạt động chính gồm xây dựng và triển khai được thực hiện thông qua GitHub Actions:

1. Nhà phát triển commit mã nguồn ứng dụng tới Source Code Repository. Trong bài viết này, chúng ta sẽ tận dụng một kho lưu trữ của AWS CodeCommit.
2. Hệ thống quản lý mã nguồn (SCM) kích hoạt AWS CodePipeline - dịch vụ điều phối và quản lý CI/CD pipeline.
3. AWS CodePipeline thực hiện giai đoạn Build (xây dựng/đóng gói), quá trình này được tích hợp với GitHub Actions để tạo container image từ mã nguồn được commit. 
4. Khi container image được xây dựng thành công, AWS CodeBuild cùng với GitHub Actions, đẩy image tới Amazon Elastic Container Registry (ECR) cho mục đích lưu trữ và đặt tên phiên bản.
5. Giai đoạn Approval (phê duyệt) được bao gồm trong pipeline, nó cho phép nhà phát triển đánh giá kết quả của các hoạt động trước đó, nếu kết quả hợp lệ thì sẽ phê duyệt để pipeline đi tiếp tới bước triển khai.
6. Sau khi nhận được thông tin phê duyệt, AWS CodePipeline sẽ chuyển tới giai đoạn Deploy (triển khai), ở đây, GitHub Actions được sử dụng để thực thi các câu lệnh triển khai bằng Helm. 
7. Trong giai đoạn triển khai, AWS CodeBuild sử dụng GitHub Actions để cài đặt ứng dụng Helm trên Amazon Elastic Kubernetes Service (EKS), tận dụng Helm charts để triển khai.
8. Ứng dụng được triển khai lúc này sẽ hoạt động trên Amazon EKS và có thể truy cập được thông qua Application Load Balancer - được cung cấp, cài đặt tự động.

## Bước chuẩn bị

Để hoàn thành các bước trong bài viết này, bạn sẽ cần chuẩn bị:

- Tài khoản [AWS](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all)
- Một [EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) với phần bổ trợ đã được cấu hình [AWS Fargate Profile](https://docs.aws.amazon.com/eks/latest/userguide/fargate-profile.html) và [ALB Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.7/). Nếu cần, bạn có thể tận dụng công cụ dòng lệnh [eksdemo](https://github.com/awslabs/eksdemo) để tạo môi trường EKS environment.
- Môi trường phát triển với các công cụ sau:
  - [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
  - [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  - [helm](https://helm.sh/docs/intro/install/#through-package-managers)

Các công cụ tiện ích `awscli` và `eksctl` được yêu cầu để truy cập tài khoản AWS. Khi sử dụng AWS CLI, đảm bảo rằng bạn có cấu hình credential, tham khảo [tài liệu sử dụng AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).

## Hướng dẫn chi tiết

### Triển khai các thành phần cơ bản

Để bắt đầu bạn sẽ cần triển khai stack AWS CloudFormation để tạo các tài nguyên nền tảng của nhà phát triển, ví dụ như kho lưu trữ mã nguồn trên CodeCommit, dự án CodeBuild, pipeline trong CodePipeline để điều phối phát hành ứng dụng qua nhiều các giai đoạn. Nếu bạn có hứng thú tìm hiểu về các tài nguyên được triển khai này, bạn có thể tải về các ví dụ và xem nội dung này.

Ngoài ra, để sử dụng GitHub Action trong AWS CodeBuild, điều này yêu cầu xác thực dự án AWS CodeBuild với GitHub bằng [access token](https://docs.aws.amazon.com/codebuild/latest/userguide/access-tokens.html#access-tokens-github) - xác thực với GitHub là bắt buộc để đảm bảo [truy cập bền vững(consitent access)](https://docs.aws.amazon.com/codebuild/latest/userguide/action-runner-buildspec.html#action-runner-connect-source-provider) và không bị giới hạn truy cập bởi GitHub.

1. Đầu tiên, thiết lập các biến môi trường sẽ được sử dụng trong quá trình cấu hình lên cơ sở hạ tầng:

```bash
export CLUSTER_NAME=<cluster-name>
export AWS_REGION=<cluster-region>
export AWS_ACCOUNT_ID=<cluster-account>
export GITHUB_TOKEN=<github-pat>
```

Trong các câu lệnh trên, thay thế *cluster-name* bằng tên của EKS cluster, *cluster-region* bằng AWS region mà EKS cluster được cài đặt, *cluster-account* bằng AWS account ID của bạn(12 chữ số), và *github-pat* bằng GitHub Personal Access Token(PAT).

2. Sử dụng AWS CloudFormation template để triển khai stack, [đường dẫn CloudFormation template](https://d2908q01vomqb2.cloudfront.net/artifacts/DevOpsBlog/devops-2406/gha.yaml), sử dụng AWS CLI:

```bash
aws cloudformation create-stack \
  --stack-name github-actions-demo-base \
  --region $AWS_REGION \
  --template-body file://gha.yaml \
  --parameters ParameterKey=ClusterName,ParameterValue=$CLUSTER_NAME \
               ParameterKey=RepositoryToken,ParameterValue=$GITHUB_TOKEN \
  --capabilities CAPABILITY_IAM && \
echo "Waiting for stack to be created..." && \
aws cloudformation wait stack-create-complete \
  --stack-name github-actions-demo-base \
  --region $AWS_REGION
```

3. Khi sử dụng AWS CodeBuild/GitHub Actions để triển khai ứng dụng lên Amazon EKS, bạn sẽ cần biết service role được liên kết với dự án bằng cách thêm IAM principal để truy cập vào [aws-auth](https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html) config-map của Cluster hoặc sử dụng [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)(đề xuất). Service role của CodeBuild được tạo trước đó và ARN của role có thể được truy xuất thông tin bằng câu lệnh sau:

```bash
aws cloudformation describe-stacks --stack-name github-actions-demo-base \
--query "Stacks[0].Outputs[?OutputKey=='CodeBuildServiceRole'].OutputValue" \
--region $AWS_REGION --output text
```

### Clone mã nguồn từ CodeCommit Repository

Tiếp theo, bạn sẽ tạo một ứng dụng python flask đơn giản. Đóng gói `helm chart` cũng sẽ là yêu cầu bắt buộc để triển khai ứng dụng và mã nguồn của helm chart cũng được lưu trữ trên kho mã nguồn ở AWS CodeCommit. Bắt đầu clone Code Commit repository bằng các lệnh sau:

1. Cấu hình git client để có thể sử dụng trình trợ giúp thông tin xác thực AWS CLI CodeCommit (credential helper).  
  - Với máy tính chạy hệ điều hành dựa trên UNIX (Linux, MacOS) làm theo hướng dẫn [này](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html)
  - Với máy tính chạy hệ điều hành Windows sử dụng hướng dẫn [này](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-windows.html)

2. Lấy thông tin URL để clone bằng câu lệnh sau:

  ```bash
  export CODECOMMIT_CLONE_URL=$(aws cloudformation describe-stacks \
  --stack-name github-actions-demo-base \
  --query "Stacks[0].Outputs[?OutputKey=='CodeCommitCloneUrl'].OutputValue" \
  --region $AWS_REGION \
  --output text)
  ``` 

3. Clone và truy cập repository
  ```bash
  git clone $CODECOMMIT_CLONE_URL github-actions-demo && cd github-actions-demo
  ```

### Phát triển ứng dụng

Tại bước này, bạn sẽ viết mã ứng dụng và mã triển khai, nói cách khác, bạn có thể bắt đầu xây dựng ứng dụng và các manifest cần thiết sử dụng cho triển khai.

1. Tạo tệp tin `app.py`, ứng dụng "hello world" đơn giản bằng câu lệnh sau:

  ```bash
  cat << EOF >app.py
  from flask import Flask
  app = Flask(__name__)

  @app.route('/')
  def demoapp():
    return 'Hello from EKS! This application is built using Github Actions on AWS CodeBuild'

  if __name__ == '__main__':
    app.run(port=8080,host='0.0.0.0')
  EOF
  ```

2. Tạo Dockerfile trong cùng thư mục với tệp tin `app.py` bằng lệnh sau:

  ```bash
  cat << EOF > Dockerfile
  FROM public.ecr.aws/docker/library/python:alpine3.18 
  WORKDIR /app 
  RUN pip install Flask 
  RUN apk update && apk upgrade --no-cache 
  COPY app.py . 
  CMD [ "python3", "app.py" ]
  EOF
  ```

3. Sinh mã nguồn helm chart bằng lệnh sau:

  ```bash
  helm create demo-app 
  rm -rf demo-app/templates/* 
  ```

4. Tạo các tệp tin manifest cần thiết cho việc triển khai:
  - `deployment.yaml` - bao gồm khuôn mẫu để triển khai ứng dụng. Nó bao gồm trạng thái mong muốn khi triển khai và pod template (có các đặc tả về container image sử dụng, port của ứng dụng, ...)

    ```bash
    cat <<EOF > demo-app/templates/deployment.yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      namespace: {{ default "default" .Values.namespace }}
      name: {{ .Release.Name }}-deployment
    spec:
      selector:
        matchLabels:
          app.kubernetes.io/name: {{ .Release.Name }}
      replicas: 2
      template:
        metadata:
          labels:
            app.kubernetes.io/name: {{ .Release.Name }}
        spec:
          containers:
          - image: {{ .Values.image.repository }}:{{ default "latest" .Values.image.tag }}
            imagePullPolicy: {{ .Values.image.pullPolicy}}
            name: demoapp
            ports:
            - containerPort: 8080
    EOF
    ```

  - `service.yaml` - Mô tả đối tượng service trong Kubernetes và chỉ ra cách để truy cập vào danh sách các pod chạy ứng dụng. Service hoạt động như một bộ cân bằng tải (load balancer) để điều hướng các yêu cầu từ phía bên ngoài tới pod trong Kubernetes. Có nhiều loại service theo các tình huống khác nhau: ClusterIP, NodePort, hoặc LoadBalancer.
    
    ```bash
    cat <<EOF > demo-app/templates/service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      namespace: {{ default "default" .Values.namespace }}
      name: {{ .Release.Name }}-service
    spec:
      ports:
        - port: {{ .Values.service.port }}
          targetPort: 8080
          protocol: TCP
      type: {{ .Values.service.type }}
      selector:
        app.kubernetes.io/name: {{ .Release.Name }}
    EOF
    ```

  - `ingress.yaml` - Định nghĩa ingress rules cho phép truy cập ứng dụng từ phía bên ngoài cụm Kubernetes. Tệp tin này liên kết các yêu cầu HTTP và HTTPS tới service trong cụm Kubernetes, cho phép lưu lượng truy cập ở bên ngoài tới được dịch vụ, ứng dụng mong muốn.

    ```bash
    cat <<EOF > demo-app/templates/ingress.yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      namespace: {{ default "default" .Values.namespace }}
      name: {{ .Release.Name }}-ingress
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
    spec:
      ingressClassName: alb
      rules:
        - http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: {{ .Release.Name }}-service
                  port:
                    number: 8080
    EOF
    ```
  
  - `vaues.yaml` - Tệp tin này cung cấp các giá trị cấu hình mặc định cho Helm chart. Tệp tin này là quan trọng cho việc tùy chỉnh Helm chart khớp với yêu cầu hoặc kịch bản triển khai cho các môi trường khác nhau. Manifest dưới dây được giả định rằng Kubernetes namespace là `default`, sẽ được sử dụng làm [namespace selector](https://docs.aws.amazon.com/eks/latest/userguide/fargate-profile.html#fargate-profile-components) cho cấu hình Fargate.  

    ```bash
    cat <<EOF > demo-app/values.yaml
    ---
    namespace: default
    replicaCount: 1
    image:
      pullPolicy: IfNotPresent
    service:
      type: NodePort
      port: 8080
    EOF
    ``` 

## Tổng quan về CI/CD Pipeline

- Một CI/CD thông thường bao gồm các giai đoạn: source(cung cấp mã nguồn), build(xây dựng), test(kiểm thử), approval(phê duyệt), và deploy(triển khai).
- Trong bài viết này, AWS CodeBuild được sử dụng trong giai đoạn build và deploy. AWS CodeBuild sử dụng những tệp tin đặc tả về các hành động bên trong pipeline được gọi là **buildspec**
- Một buildspec là tập hợp của các [phase](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html#build-spec.phases) liên quan tới các cấu hình được viết dưới dạng YAML để CodeBuild có thể đọc hiểu và sử dụng trong lúc thực thi tự động.

Dưới dây, bạn sẽ học cách định nghĩa buildspec để build và deploy ứng dụng tới Amazon EKS bằng cách tận dụng [GitHub action runner quản lý bởi AWS trong AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/action-runner-buildspec.html#action-runner-connect-source-provider)

### Định nghĩa GitHub Actions trong AWS CodeBuild

Mỗi *phase* trong *buildspec* có thể bao gồm nhiều các *step* và mỗi *step* có thể thực thi các câu lệnh hoặc thực thi một GitHub Action. Mỗi *step* khi được thực thi, sẽ tương ứng là một tiến trình và filesystem của *step* đó. Một *step* tham chiếu tới một GitHub Action bằng cách chỉ định chỉ thị *[uses](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsuses)* và chỉ thị tùy chọn *[with](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepswith)* để truyền các tham số bắt buộc của Action. Theo một cách khác, một *step* có thể chỉ định các câu lệnh thông qua chỉ thị *[run](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsrun)*. Cần lưu ý rằng, bởi vì mỗi một *step* sở hữu tiến trình riêng, nên các thay đổi về biến môi trường sẽ không được duy trì giữa các *step*.

Để truyền biến môi trường giữa các step trong giai đoạn build, bạn cần gán giá trị cho một biến môi trường mới hoặc đã tạo trước đó và ghi chúng vào [tệp tin môi trường GITHUB_ENV](https://docs.github.com/en/actions/learn-github-actions/variables#passing-values-between-steps-and-jobs-in-a-workflow). Hơn nữa, các biến môi trường này cũng có thể  được truyền giữa các giai đoạn trong CodePipeline bằng cách sử dụng kỹ thuật [exported variables directive](https://docs.aws.amazon.com/codebuild/latest/userguide/action-runner-buildspec.html#exported-env-variable-sample).

### Đặc tả trong AWS CodeBuild - Giai đoạn đóng gói/xây dựng

Tại bước này, bạn sẽ tạo một tệp tin `buildspec-build.yml` ở thư mục gốc của repository. Trong buildspec dưới đây, chúng ta sẽ tận dụng GitHub actions trong AWS CodeBuild để xây dựng container image và đẩy image này lên ECR. Action được sử dụng trong buildspec gồm:
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials): Việc truy cập API của AWS yêu cầu action phải được xác thực bằng AWS [credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/security-creds.html). Mặc định, quyền gán cho role của dịch vụ CodeBuild có thể sử dụng để thêm các API action cho quá trình xây dựng. Tuy nhiên, khi sử dụng một GitHub action trong CodeBuild, credentials từ role của dịch vụ CodeBuild cần duy trì khả năng sử dụng cho các action liên tiếp nhau(ví dụ: đăng nhập ECR, đẩy image lên ECR). Các action này cho phép tận dụng credential của role có chứa dịch vụ CodeBuild cho các action kế tiếp.
- [aws-actions/amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login): Đăng nhập ECR registry với thông tin credential ở bước trước.

```bash
version: 0.2
env:
  exported-variables:
    - IMAGE_REPO
    - IMAGE_TAG
phases:
  build:
    steps:
      - name: Get CodeBuild Region
        run: |
          echo "AWS_REGION=$AWS_REGION" >> $GITHUB_ENV
      - name: "Configure AWS credentials"
        id: creds
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-region: ${{ env.AWS_REGION }}
          output-credentials: true
      - name: "Login to Amazon ECR"
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      - name: "Build, tag, and push the image to Amazon ECR"
        run: |
          IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
          docker build -t $IMAGE_REPO:latest .
          docker tag $IMAGE_REPO:latest $IMAGE_REPO:$IMAGE_TAG
          echo "$IMAGE_REPO:$IMAGE_TAG"
          echo "IMAGE_REPO=$IMAGE_REPO" >> $GITHUB_ENV
          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV
          echo "Pushing image to $REPOSITORY_URI"
          docker push $IMAGE_REPO:latest
          docker push $IMAGE_REPO:$IMAGE_TAG
```

Trong buildspec ở phía trên, các biến `IMAGE_REPO` và `IMAGE_TAG` được thiết lập là những biến môi trường, cho phép chúng có thể được sử dụng ở giai đoạn tiếp theo.

### Đặc tả trong AWS CodeBuild - Giai đoạn triển khai

Trong giai đoạn triển khai, bạn sẽ tận dụng AWS CodeBuild để triển khai các helm manifest tới EKS bằng cách sử dụng action từ cộng động [bitovi/deploy-eks-helm](https://github.com/bitovi/github-actions-deploy-eks-helm). Tiếp theo, action [alexellis/arkade-get](https://github.com/alexellis/arkade-get) được sử dụng để cài đặt `kubectl`, công cụ này sẽ được sử dụng ở bước sau để lấy thông tin về URL của ứng dụng.

Tạo một tệp tin với tên `buildspec-deploy.yml` ở thư mục gốc của repository với nội dung sau:

```bash
version: 0.2
env:
  exported-variables:
   - APP_URL
phases:
  build:
    steps:
      - name: "Get Build Region"
        run: |
          echo "AWS_REGION=$AWS_REGION" >> $GITHUB_ENV        
      - name: "Configure AWS credentials"
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-region: ${{ env.AWS_REGION }}
      - name: "Install Kubectl"
        uses: alexellis/arkade-get@23907b6f8cec5667c9a4ef724adea073d677e221
        with:
          kubectl: latest
      - name: "Configure Kubectl"
        run: aws eks update-kubeconfig --name $CLUSTER_NAME
      - name: Deploy Helm
        uses: bitovi/github-actions-deploy-eks-helm@v1.2.7
        with:
          aws-region: ${{ env.AWS_REGION }}
          cluster-name: ${{ env.CLUSTER_NAME }}
          config-files: demo-app/values.yaml
          chart-path: demo-app/
          values: image.repository=${{ env.IMAGE_REPO }},image.tag=${{ env.IMAGE_TAG }}
          namespace: default
          name: demo-app
      - name: "Fetch Application URL"
        run: |
          while :;do url=$(kubectl get ingress/demo-app-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' -n default);[ -z "$url" ]&&{ echo "URL is empty, retrying in 5 seconds...";sleep 5;}||{ export APP_URL="$url";echo "APP_URL set to: $APP_URL";break;};done;echo "APP_URL=$APP_URL">>$GITHUB_ENV 
```

Tới bước này, bạn cần lưu ý rằng cấu trúc thư mục của ứng dụng sẽ như sau:

```text
├── Dockerfile
├── app.py
├── buildspec-build.yml
├── buildspec-deploy.yml
└── demo-app
├── Chart.yaml
├── charts
├── templates
│ ├── deployment.yaml
│ ├── ingress.yaml
│ └── service.yaml
└── values.yaml
```

Bây giờ, bạn sẽ đẩy các tệp tin này tới remote repository bằng các câu lệnh dưới đây:

```bash
git add -A && git commit -m "Initial Commit"
git push --set-upstream origin main
```

Bây giờ, hãy xác nhận ứng dụng đã được triển khai thông qua truy cập load balancer URL. Điều hướng trình duyệt tới [CodePipeline console](https://us-west-1.console.aws.amazon.com/codesuite/codepipeline/pipelines/github-actions-demo-pipeline/view?region=us-west-1). Trong pipeline có một giai đoạn phê duyệt thủ công và yêu cầu nhà vận hành đánh giá và phê duyệt bản phát hành này để pipeline có thể tiếp tục hoàn thành triển khai ứng dụng. Khi pipeline chạy thành công, URL của ứng dụng được triển khai có thể lấy từ kết quả của lần thực thi pipeline này.

### Nghiệm thu ứng dụng

1. Nhấp chuột vào execution ID. Thông tin tổng quan về lần thực thi gần nhất sẽ xuất hiện.

{{< image src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2024/04/29/gha_review_1.jpg" caption="Ảnh 2: Thông tin tổng quan của execution ID trong CodePipeline Console" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

2. Bên dưới tab Timeline, chọn 'Build' action.

{{< image src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2024/04/29/gha_review_2.jpg" caption="Ảnh 3: Điều hướng tới tab timeline và xem thông tin của giai đoạn stage" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

3. Sao chép URL của load balancer cho ứng dụng trong phần 'Output varialbes'.

{{< image src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2024/04/29/gha_review_3.jpg" caption="Ảnh 4: Sao chép APP_URL từ Output Variables trong Deploy action" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

4. Mở URL trên trình duyệt và chờ đợi dòng chữ như dưới đây.

{{< image src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2024/04/29/gha_review_4.jpg" caption="Ảnh 5: Kết quả hiển thị của ứng dụng được triển khai trên Amazon EKS" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Bạn cũng có thể xem lại log của quá trình xây dựng cùng với hoạt động của GitHub action trong [console](https://us-west-1.console.aws.amazon.com/codesuite/codebuild/projects?region=us-west-1) của AWS CodeBuild.

## Dọn dẹp môi trường

Để tránh tổn thất chi chí trong tương lai, bạn nên dọn dẹp các tài nguyên đã tạo ra:
  - Xóa ứng dụng bằng cấu lệnh helm, điều này cũng sẽ đi kèm với việc xóa ALB:
    ```bash
    helm uninstall demo-app
    ```

  - Xóa CloudFormation stack(github-actions-demo-base) bằng câu lệnh sau:
    ```bash
    aws cloudformation delete-stack \
        --stack-name github-actions-demo-base \
        -–region $AWS_REGION
    ```

## Tổng kết

Trong hướng dẫn này, bạn đã học cách tận dụng sức mạnh từ sự kết hợp các GitHub Action và AWS CodeBuild để đơn giản hóa và tự động triển khai ứng dụng Python trên Amazon EKS. Hướng tiếp cận này không chỉ làm mượt quá trình triển khai mà còn đảm bảo rằng ứng dụng của bạn sẽ được xây dựng và triển khai một cách an toàn. Bạn có thể mở rộng pipeline với những giai đoạn thêm vào, ví dụ như kiểm thử, quét an toàn thông tin, phụ thuộc vào yêu cầu của dự án. Ngoài ra, giải pháp này cũng có thể áp dụng cho các ngôn ngữ lập trình khác.