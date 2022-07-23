---
layout: post
current: post
cover:  assets/built/images/background/terraform.png
navigation: True
title: Terraform Tips 5 - Import
date: 2022-07-23 00:00:00
tags: [devops, terraform]
class: post-template
subclass: 'post tag-devops'
author: HeuristicWave
---

Terraform 더 익숙하게 5 - Import


## Intro

IaC를 도입하기 위해 구축 단계부터 코드로 인프라를 작성할 수도 있지만, 기 구축된 인프라를 코드화할 수도 있습니다.
이때 사용하는 Terraform의 기능이 바로 `Import`입니다.

하지만 저는 구축 단계에도 종종 `Import` 기능을 활용합니다. Terraform으로 코드를 작성하기 위해 [registry.terraform.io](https://registry.terraform.io/providers/hashicorp/aws/latest/docs )에서
가이드 하는 대로 코드를 작성하는 것이 생각보다 어려운 작업이기 때문이죠. 🥲

그래서 저는 먼저 구축하고자 하는 인프라를 콘솔상에서 구성한 다음, 구축에 필요한 Attribute 들을 파악합니다.
그다음 `Import`를 사용해 동작하는 IaC 코드를 얻고 수정합니다.
즉, 저는 `Import` 기능을 Cheat Sheet처럼 사용하고 있습니다. 😅

이번 포스팅에서는 실제 제가 Cheat Sheet으로 활용하는 *'Import 시나리오'*를 통해 `Import`를 학습해 보겠습니다. 

<br>

## ⏮ 조립은 분해의 역순

AWS Systems Manager의 인스턴스 운영 자동화를 위한 State Manager 기능을 사용하기 위해 [문서](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_association )를
확인해 보았지만, 다음과 같은 사용법 만이 기재되어 있습니다.

```terraform
resource "aws_ssm_association" "example" {
  name = aws_ssm_document.example.name

  targets {
    key    = "InstanceIds"
    values = [aws_instance.example.id]
  }
}
```

해당 코드를 apply 해도 무수한 Error만 만날 뿐 빠르게 진도가 나가지 않기에, 우선 AWS 웹 콘솔을 활용해 인스턴스 운영 자동화를 위한 State Manager 기능을 구현해 두었습니다.

### 0️⃣ 준비 작업

이번 포스팅의 작업 공간(~/terraform)을 생성하고 해당 위치에서 아래 코드 블록을 터미널에 복사합니다. (리소스가 위치한 리전 명에 맞게 세팅해 주세요)

```shell
cat <<EOF > provider.tf
provider "aws" {
  region  = "ap-northeast-2"
}
EOF
```
이후, `terraform init` 명령어를 실행시켜주세요.

### 1️⃣ Skeleton Code 작성

웹 콘솔로 작업한 State Manager를 코드화하기 위해 아래와 같이 Skeleton Code를 작성합니다.
(import 후, 생성되는 `*.tfstate`를 담는 일종의 빵틀을 제작하는 단계입니다.)

```shell
cat <<EOF > main.tf
resource "aws_ssm_association" "copycat" {}
EOF
```

### 2️⃣ Import Configuration

사용법(`terraform import [options] ADDRESS ID`)에 따라 아래 명령어를 실행시키면 **root directory**에 미리 생성된 인프라가 `*.tfstate` 파일에 담깁니다.

```shell
$ terraform import aws_ssm_association.copycat <Association ID>
```

생성된 `*.tfstate` 파일을 확인하면 json 형태로 `aws_ssm_association` resource block에 작성해야 하는 각종 Config 값들을 알 수 있습니다.
그러나, `show` 명령어를 사용해 HCL Syntax에 맞춰 **human-readable**한 형태로 출력합니다.

```shell
$ terraform show -no-color > main.tf
```

본래 `-no-color` 옵션은 coloring 작업을 비활성화하지만, Editor에 format 맞추기 위해 필수적으로 해당 옵션을 사용합니다.

### 3️⃣ Modify Arguments

이제서야 얼추 모양을 갖춘 `main.tf`의 **resource block**에는 리소스가 인프라에 **반영된 이후 단계에 생성되는 각종 result** 값(`arn`, `association_id`, `id`)
이 포함되어 있습니다. 해당 값들은 `terraform plan` 명령어를 수행해 Error 메시지에 명시되므로 지워야 하는 Arguments들을 찾아 코드를 수정합니다.

이때, Instancd Id, IAM Role ARN 등과 같이 **절대적인 값**도 **재사용 가능한 변수**로 처리하는 것이 좋습니다.
해당 작업을 마치고 나면, `plan`, `apply` 명령어를 수행하여 다음 메시지를 얻으면 정상적으로 Import 작업이 완료된 것입니다.

> Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
   
<br>

## 🥵 Import into Module 

지금까지 학습한 절차는 단순 **Resource**에 Import 시키므로 비교적 수월한 과정이었습니다.

> Resource configured with count <br>
> ➡️ `terraform import 'aws_instance.baz[0]' i-abcd1234` <br>
> Resource configured with for_each <br>
> ➡️ `terraform import 'aws_instance.baz["example"]' i-abcd1234` <br>
> Module <br>
> ➡️ `terraform import module.foo.aws_instance.bar i-abcd1234`

Module을 Import 하는 절차도 Resource와 동일하지만, 명백한 **한계점**이 있습니다.

흔하게 사용되는 [ec2-instance](https://registry.terraform.io/modules/terraform-aws-modules/ec2-instance/aws/latest) 모듈을 사용한다고 가정하겠습니다.
Resource 때와 동일하게 아래와 같은 Skeleton Code를 작성하고 Import 명령어를 수행하는 부분은 동일합니다. (1️⃣ & 2️⃣ 과정 동일)

```terraform
module "ec2_instance" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "~> 3.0"

  name = "single-instance"
}
```

이때, `foo`는 `ec2_instance`가 되고 `bar`는 해당 모듈의 **aws_instance**에서 정의한 `this`가 됩니다.

문제는 **3️⃣ Modify Arguments** 단계에서 발생합니다. Single Resource에서는 `plan`을 통해 수정해야 하는 **Arguments**들을 알 수 있지만, 
module에서 skeleton code 이외에 더 기재해야 하는 variable 값들을 알 수 없습니다.
즉, module import 이후 생성되는 `*.tfstate` 파일의 **json 값을 일일이 확인**하여 module의 **input parameter에 해당하는 값**들을 알아내 **하나씩 다 기재**하는 방법 외에는 
온전하게 import 명령어를 사용할 수 없습니다.

Reverse Engineering으로 원래의 코드를 완벽하게 재현하기 어려운 것처럼, 구성이 복잡한 모듈은 Reverse Terraforming이 매우 어렵습니다.

<br>

## 🌏 Reverse Terraform Open Source

**2️⃣ Import Configuration** 단계에서 수행한 작업은 [Terraformer](https://github.com/GoogleCloudPlatform/terraformer )(작성 시점 기준 ★ 8.1k),
[Terraforming](https://github.com/dtan4/terraforming )(업데이트 종료) 등과 같은 오픈소스 도구를 활용할 수도 있습니다.

**⚠️ 고려 사항**<br>
**Terraformer**는 [AWS configuration Profiles Select](https://github.com/GoogleCloudPlatform/terraformer/blob/master/docs/aws.md#profiles-support )와 [Attribute filters](https://github.com/GoogleCloudPlatform/terraformer/blob/master/docs/aws.md#attribute-filters )과 같은 편리한 기능들을 제공합니다. <br>
그러나 [terraformer AWS 리소스 지원 범위](https://github.com/GoogleCloudPlatform/terraformer/blob/master/docs/aws.md#supported-services )에서도 확인할 수 있다시피,
위에서 작업한 `aws_ssm_association`과 같이 지원하지 않는 리소스들도 존재합니다.

<br>

## Outro

마지막으로 Import Workflow를 다시 한번 정리하면서 마치겠습니다.

1. Import 대상(이미 프로비저닝 된 인프라)이 되는 Skeleton Code 작성
2. Write Config : Import & Show 명령어 수행
3. Modify Arguments
4. Plan & Apply

지금까지 테라폼 더 익숙하게 Import 편을 읽어주셔서 감사합니다! 잘못된 내용은 지적해 주세요! 😃

<br>

**📚 References**

- Terraform Documentation [Import Command](https://www.terraform.io/cli/commands/import)
- Hashicorp Tutorial 문서 [Import Terraform Configuration](https://learn.hashicorp.com/tutorials/terraform/state-import?in=terraform/state)


---

{% include terraform_tips.html %}

<br>