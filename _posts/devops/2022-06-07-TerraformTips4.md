---
layout: post
current: post
cover: assets/built/images/background/terraform.png
navigation: True
title: Terraform Tips 4 - Move (Refactoring)
date: 2022-06-07 00:00:00
tags: [devops, terraform]
class: post-template
subclass: 'post tag-devops'
author: HeuristicWave
---

Terraform 더 익숙하게 4 - Move (Refactoring)


## Intro

[지난 3편](https://heuristicwave.github.io/TerraformTips3 )에서는 *코드의 변경 없이* 형상을 유지하는 기법 중 하나인 `Refresh`와 `Replace`에 대하여 알아보았습니다.
4편에서는 *코드를 변경(Refactoring)* 할 때 형상을 유지하는 방법인 `Move`와 관련된 기능들을 소개합니다.

<br>

## 🤹 Moving Resources

Terraform으로 정의한 인프라는 `terraform.tfstate`에 기록되며, real-world의 객체는 특정 [리소스 주소](https://www.terraform.io/cli/state/resource-addressing )와 연결되어 있습니다.
그래서 정의한 인프라 코드를 변경 후, 적용하면 real-world의 객체와 상태가 변경됩니다.
코드로 정의된 실제 인프라를 운영하고 있다면, 코드 리팩토링 시 발생하는 리소스 변경 지점을 `State` 명령어를 활용해 해소해야 합니다.

**Commands**

- `state mv` : real-world 객체와 연결된 리소스 주소를 변경
- `state rm` : real-world 객체를 파괴하지 않고 코드로 정의한 리소스 관리 대상 제거
- `state replace-provider` : 재생성 없이, 새로운 provider에 기존 리소스 전

### 🛠 state mv

위 3가지 명령어 중에서도 가장 활용도가 높은 `state mv` 명령어의 예시들을 확인해 보겠습니다. 

1. [리소스 이름 재정의](https://www.terraform.io/cli/commands/state/mv#example-rename-a-resource) : 
   리팩토링 과정에서 정의된 리소스 명을 변경하는 경우 <br> 
   `terraform state mv {ResourceType}.{ExistingName} {ResourceType}.{ChangedName}`
2. [리소스를 모듈로 이동](https://www.terraform.io/cli/commands/state/mv#example-move-a-resource-into-a-module) : 
   루트에 위치한 리소스를 child 모듈에 포함시켜 리팩토링 하는 경우 <br>
   `terraform state mv {Type}.{Name} module.{ModuleName}.{Type}.{Name}`
3. [모듈을 다른 모듈로 이동](https://www.terraform.io/cli/commands/state/mv#example-move-a-module-into-a-module) <br>
   `terraform state mv module.{ModuleName} module.{ParentModuleName}.module.{ModuleName}`
4. [meta-argument `for_each`, `count`로 정의된 특정 리소스 교체](https://www.terraform.io/cli/commands/state/mv#example-move-a-particular-instance-of-a-resource-using-count) <br>
   `terraform state mv {ResourceType}.{Before}[0] {ResourceType}.{After}[0]`
   
<br>

## 🏃️ Moved statements

[지난 3편](https://heuristicwave.github.io/TerraformTips3 )에서 `refresh`와 `taint`가 가진 한계점으로 인하여
`--refresh-only`와 `replace`가 나왔듯이, `state mv`도 한계점이 있습니다.

> The `terraform taint` command informs Terraform that a particular object has become degraded or damaged. <br>

### [Problems with `terraform state mv`](https://youtu.be/bDgoGBusX0k?t=178)

위 링크로 첨부한 Terraform 1.1 버전이 출시하면서 발표한 `Move` 기능에 대한 발표 자료를 보면 아래 3가지 이유로 한계점을 다룹니다.

1. Risky and error prone
2. Terraform Cloud users couldn't refactor within core workflows
3. Module authors couldn't coordinate changes themselves
   
3가지의 한계점이 언급되었지만, 결국 `refresh` 때와 마찬가지로 `plan`을 통한 예측 과정이 없는 동일한 이유로 위와 같은 문제가 발생한다는 사실을 알 수 있습니다. 

### [Refactoring](https://www.terraform.io/language/modules/develop/refactoring)

앞서 언급한 한계점들은 `plan` 기능이 없는 `mv` 명령어 대신 기존의 테라폼 문법에 `moved` block이 추가되며 해소되었습니다.
아래 예시는 [공식 문서](https://www.terraform.io/language/modules/develop/refactoring#moved-block-syntax )에 기재된 예제입니다.

```terraform
locals {
  instances = tomap({
    big = {
      instance_type = "m3.large"
    }
    small = {
      instance_type = "t2.medium"
    }
  })
}

resource "aws_instance" "a" {
  for_each = local.instances

  instance_type = each.value.instance_type
  # (other resource-type-specific configuration)
}

moved {
  from = aws_instance.a
  to   = aws_instance.a["small"]
}
```

`moved` block에 변경 전후 상태를 선언하여, 기존 상태를 바꾸는 명령조차도 코드로 선언하여 상태를 바꾸는 행위를 코드화했습니다.

### ⁉️ 고려사항

클라우드 환경에서 운영을 하다 보면 최적화 과정 중 리소스 스펙(인스턴스 종류, 타입)이 자주 변경됩니다.
그럼 아래와 같은 `moved` 블록이 체인과 같이 길어지게 됩니다.

```terraform
# Block 1
moved {
  from = aws_instance.a
  to   = aws_instance.b
}

# Block 2
moved {
  from = aws_instance.b
  to   = aws_instance.c
}
```

이는 오히려 사용하지 않는 혹은 *중복된 코드를 지우고 로직을 이해하기 쉽게 디자인*해야 하는 **리팩토링과는 멀어지게** 됩니다.
그러므로 AWS 리소스의 경우 **단순 스펙 변경**과 같은 작업은 `Launch templates`을 사용하는 게 좋습니다.
이처럼 AWS 서비스 내에서 **형상 관리** 기능을 제공하는 서비스를 활용해, moved 블록을 생성하는 작업을 최소화해야 합니다.



<br>

## Outro

지금까지 리팩토링 작업을 위해 필수적으로 사용되는 `state mv`와 `moved` syntax 사용법을 알아보았습니다.
`Move`의 탄생 과정에서 선언형 도구인 테라폼의 목적에 맞게 진화해나가는 모습과 초기 설계의 중요성을 고민해 볼 수 있었던 좋은 기회였습니다.
여담으로 저는 테라폼 버전이 0.12일 때 다루기 시작했는데, 2년 만에 버전 1.2.2에 이르며 정착해나가는 과정을 보니
사용자도 IaC 도구의 철학을 이해하며 함께 성장하는 기분에 감격스럽습니다.

지금까지 테라폼 더 익숙하게 Move(Refactoring) 편을 읽어주셔서 감사합니다! 잘못된 내용은 지적해 주세요! 😃

---

{% include terraform_tips.html %}
