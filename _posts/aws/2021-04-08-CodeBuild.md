---
layout: post
current: post
cover:  assets/built/images/post/CodePipeline.png
navigation: True
title: ECR CodePipeline with Terraform Ⅱ
date: 2021-04-08 00:00:00
tags: [aws, devops, terraform]
class: post-template
subclass: 'post tag-aws'
author: HeuristicWave
---

Terraform으로 ECR 파이프라인 구축하기 2 (ECR, CodeBuild, IAM)

2편에서는 **ECR**과 **CodeBuild**를 생성하고 **IAM 역할, 정책을 부여**하는 법을 학습합니다.

## ECR
ECR 역시 [공식 문서](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecr_repository )에서 사용방법을 확인합니다.
공식문서에서 `image_scanning_configuration` config를 사용하면 취약점 스캔이 가능하다 설명되어 있지만, 필요하지 않기 때문에 제외하겠습니다.
더불어, output도 함께 작성하겠습니다.
```shell
cat <<EOF > ecr.tf
resource "aws_ecr_repository" "image_repo" {
  name                 = var.image_repo_name
  image_tag_mutability = "MUTABLE"
}

output "image_repo_url" {
  value = aws_ecr_repository.image_repo.repository_url
}

output "image_repo_arn" {
  value = aws_ecr_repository.image_repo.arn
}
EOF
```
이어서 `ecr.tf`에서 변수로 사용하기 위한 **var.image_repo_name** 부분이 작동하도록 1편에서 작성한 `variables.tf` 아래 값을 추가합니다.

✅ 편의상 이번 단계에 필요한 variable을 함께 포함했습니다.
```
variable "image_repo_name" {
  description = "Image repo name"
  type        = string
}

variable "container_name" {
  description = "Container Name"
  default     = "my-container"
}

variable "source_repo_branch" {
  description = "Source repo branch"
  type        = string
}
```
ecr 작성을 완료햇으니 `plan, apply` 명령어를 차례로 입력해 인프라를 생성하고 `terraform state list`명령어나 [콘솔](https://console.aws.amazon.com/ecr/home )에서 생성된 인프라를 확인합니다.

## CodeBuild
CodeBuild를 사용하기 위해 [Terraform 도큐먼트](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/codebuild_project )에서 사용법을 확인합니다.
기존까지의 작업과는 달리 상당히 어려워 보입니다. 그러나 쓱 훝어보면 크게 4가지(bucket, IAM Role과 Policy, Codebuild)로 정리됩니다.

### Bucket
도큐먼트와 같이 우선적으로 S3를 생성합니다. bucket의 이름은 선택이지만, 여러개의 버킷을 가지고 있는 저는 식별을 위해 이름을 부여했습니다.
```shell
cat <<EOF > codebuild.tf
resource "aws_s3_bucket" "artifact_bucket" {
  bucket = "ecr-pipeline"
}
EOF
```

### IAM Role
도큐먼트를 따라 AssumeRole을 사용합시다. <br>
➕ S3을 만들때 사용한 `codebuild.tf`에 아래 코드를 추가합니다.
```
resource "aws_iam_role" "codebuild_role" {
  name = "terraform-codebuild"
  assume_role_policy = <<EOF
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": {
            "Service": "codebuild.amazonaws.com"
         },
         "Action": "sts:AssumeRole"
      }
   ]
}
EOF
}
```

### IAM Policy
정책은 IAM 콘솔에서 기존에 만들어진 정책을 사용할 수도 있지만, 아래와 같이 직접 작성할 수도 있습니다.
도큐먼트에서 EC2에 대한 정책을 사용하지만, 우리는 ECR을 사용하므로 아래와 같은 정책을 사용하겠습니다.
{% gist heuristicwave/09103b88af041153ccd206ec6d51b7c1 %}

20, 30라인에서 앞서 생성한 리소스를 `${채움참조}` 문법으로 유연한 코드를 작성합니다.

🚩 이어서 생성한 **Policy를 Role에 부여**합니다. 이것 역시 `codebuild.tf`에 추가합니다.
```
resource "aws_iam_role_policy_attachment" "codebuild-attach" {
  role       = aws_iam_role.codebuild_role.name
  policy_arn = aws_iam_policy.codebuild_policy.arn
}
```

### CodeBuild
[Terraform 도큐먼트](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/codebuild_project )를 보아도 어떻게 해야 ECR에 적용시킬 수 있는지 알기 어렵습니다.
우선 CodeBuild를 이해하기 위해 [AWS docs](https://docs.aws.amazon.com/ko_kr/codebuild/latest/userguide/sample-docker.html )를 읽어봅시다.
대략 리소스 이름을 정하고, 환경을 구성하고 빌드를 하기 위한 방법을 정의해야 한다는 사실을 알 수 있습니다.
CodeBuild가 정의된 아래 코드를 활용해 `codebuild.tf`에 추가합니다.
{% gist heuristicwave/2ebf79ce3cbdf4a87657b272f9e1d994 %}

31라인이 참조하는 `buildspec.yml`을 pre_build, build, post_build에 맞춰 작성합니다.
```yaml
cat <<EOF > buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      docker: 18
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
EOF
```

지금까지 작성된 인프라를 `terraform state list`명령어를 통해 확인하면 아래와 같습니다.
```shell
❯ terraform state list
aws_codebuild_project.codebuild
aws_codecommit_repository.test
aws_ecr_repository.image_repo
aws_iam_policy.codebuild_policy
aws_iam_role.codebuild_role
aws_iam_role_policy_attachment.codebuild-attach
aws_s3_bucket.artifact_bucket
```

<details><summary markdown="span">생성한 인프라가 위와 같지 않을 경우, 👉 Click</summary>

실수로 의도치 않은 인프라가 프로비저닝 되었다면 2가지 방법을 통해 원 상태로 복구 할 수 있습니다.
1. `terraform destroy` 명령어로 특정 인프라만 되돌리거나 프로비저닝 하고싶은 경우, `-target` 옵션과 함께 resource 명으로 명령어를 작성합니다. <br>
   *예시) terraform destory -target aws_vpc.main*
2. 잘못 작성한 코드를 수정 후, `terraform apply`명령어를 적용하여 최신 상태의 인프라를 반영합니다.

</details>


<br>


---

{% include terraform.html %}

<br>