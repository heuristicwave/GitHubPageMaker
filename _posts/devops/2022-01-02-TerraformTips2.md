---
layout: post
current: post
cover:  assets/built/images/background/terraform.png
navigation: True
title: Terraform Tips 2 - Data & Index
date: 2022-01-02 00:00:00
tags: [devops, terraform]
class: post-template
subclass: 'post tag-devops'
author: HeuristicWave
---

Terraform 더 익숙하게 2 - Data & Index


## Intro

이번 포스팅은 Tip이라 하기에는 부끄러운 사소한 지식이지만, 제가 자주 실수 하는 내용이라 글로 남기게 되었습니다. **Terraform Data**를 잘 활용하면 디스크 이미지, 코드로 정의한 다양한 리소스 및 클라우드 공급자 API에서 가져온 정보들을 알 수 있습니다.
모든 `Data Sources`가 동일한 방법으로 간편하게 조회할 수 있으면 좋겠지만, 막상 사용하려고 하면 이런 저런 문제들을 만나게 됩니다.

공식문서([Tutorial : Query Data Sources](https://learn.hashicorp.com/tutorials/terraform/data-sources)) 에서도 Data 활용방법을 배울 수 있지만,
이번 포스팅에서는 3가지 예제와 함께 리소스를 Query하는 방법을 배워 보겠습니다.

<br>

## Query AMI

AWS의 최신 AMI는 주기적으로 갱신됩니다. 따라서 재사용 가능한 코드를 작성하기 위해, 항상 최신 AMI를 참조하는 코드를 작성하는데 다음과 같은 방법을 사용할 수 있습니다.

```shell
data "aws_ami" "amazon_linux" {
  most_recent = true

  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-kernel-*-hvm-*-x86_64-gp2"]
  }
}

output "name" {
  value = data.aws_ami.amazon_linux.id
}
```

위와 같은 방법으로 `filter`와 `owners` 값을 조정하며 어떤 이미지든지 id 값(output)을 얻어 낼 수 있습니다.
예를 들어 ECS와 EKS에서 Optimized AMI를 사용하는 경우, 다음과 같은 filter 값을 줄 수 있습니다.

> values = ["amzn2-ami-ecs-hvm-*-x86_64-ebs"]

## Query AZ

AWS의 리전마다 사용가능한 az가 다르기 때문에, 조금 더 유연한 코드를 작성하기 위해 다음 코드를 사용해 사용가능한 az를 검색합니다.
이후, `data.<NAME>.<ATTRIBUTE>.names` 로 사용가능한 az 값들을 확인할 수 있습니다.

```shell
$ data "aws_availability_zones" "available" {
  state = "available"
}

$ output "azs" {
 value = data.aws_availability_zones.available.names
}
```

`names`에는 사용가능한 az가 배열 형태로 들어가 있어, `names[0]`, `names[1]`과 같이 Index 값으로 특정 값을 지정할 수 있습니다.
그러나, 모든 data가 Index 값을 가지고 있는 것은 아닙니다. 

## Query vpc_id

다른 리소스와 AWS 솔루션들을 연계하기 위해서는 `vpc_id` 값이 필수적으로 들어갑니다.
vpc의 id를 구하기 위해서는 다음과 같은 방법으로 id를 조회할 수 있습니다.
(tags 값을 활용해 일종의 필터링을 사용할 수도 있습니다.)

```shell
data "aws_vpcs" "vpcs" {
    tags = {
        Name = var.vpc_name
    }
}

output "vpc_id" {
    value = data.aws_vpcs.vpcs.ids
}
```

위 코드로 다음과 같이 Output 값을 얻을 수 있지만, 하나의 `az` 값을 얻을때와 동일한 방식으로 `ids[0]` 형식으로 값을 조회하려 하면,
"This value does not have any indices." 라는 에러 메시지와 함께 출력을 지원하지 않습니다.

```shell
Changes to Outputs:
  + vpc_id = [
      + "vpc-0x1234567890",
    ]
```

도대체 무엇이 잘못된 것일까요? az와 동일한 방법으로 접근했지만, 왜 지원하지 않는지는 아직까지도 모르겠습니다...
누구 아시는 분이 있다면 알려주세요.

### count로 index 부여하기

위 문제를 해결하기 위해서는 az를 검색할 때보다는 불편하지만, `count`를 사용해 해결할 수 있습니다.

```shell
data "aws_vpc" "target" {
  count = length(data.aws_vpcs.vpcs.ids)
  id    = tolist(data.aws_vpcs.vpcs.ids)[count.index]
}

# sample code using vpc_id
resource "aws_lb_target_group" "sample_resource" {
  count = length(data.aws_vpcs.vpcs.ids)
  # Skip Config
  vpc_id      = data.aws_vpc.target[count.index].id
}
```

`aws_vpcs`가 아닌 `aws_vpc`를 추가하고 index 를 부여하기 위한 내장 함수들을 사용해 index를 부여합니다.
이후, 리소스에서 data 값들을 식별하기 위한 `count`를 기입하고 위와 같이 index 값으로 조회가 가능합니다.

<br>

## Outro

지금까지 Data를 활용해 각종 리소스들을 검색하는 방법을 알아 보았습니다.`vpc_id`도 `az`와 같이 별도의 index 과정 없이,
간편한 조회가 가능하면 좋겠습니다. (제가 아직 방법을 모르는 것일 수도 있어요!)

지금까지 테라폼 더 익숙하게 Data & Index 편을 읽어주셔서 감사합니다! 잘못된 내용은 지적해 주세요! 😃

<br>

---

{% include terraform_tips.html %}

<br>