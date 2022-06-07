---
layout: post
current: post
cover:  assets/built/images/background/self.png
navigation: True
title: Self-service Infrastructure
date: 2021-10-25 00:00:00
tags: [devops]
class: post-template
subclass: 'post tag-devops'
author: HeuristicWave
---

본 글은 [Kief Morris의 Infrastructure as Code](https://www.oreilly.com/library/view/infrastructure-as-code/9781491924334/ )와 [HashiCorp 백서](https://www.hashicorp.com/resources/unlocking-the-cloud-operating-model-provisioning )를 읽고 학습한 내용을 기반으로 작성한 글 입니다.

<br>

## 서문

IaC(Infrastructure as Code)에 관심을 갖고 공부를 하다보면 **Self-service Infra**라는 말을 자주 만나게 됩니다.
몇 개월째 와닿지 않는 개념이였지만, 최근 **키프 모리스**의 책을 다시 읽고 조금은 알게 된 거 같아 그동안 공부한 Self-service Infra에 대한 자료들을 바탕으로 작성해 보았습니다.

**Self-service Infra**에 관한 배경지식을 넓히는 데 도움이 되었으면 좋겠습니다.

<br>

## Dynamic Infrastructure

**동적 인프라**는 서버, 스토리지, 네트워크와 같은 인프라 자원을 관리할 수 있는 시스템을 말합니다.
동적 인프라의 종류로는 Public/Private 클라우드, 오픈스택을 활용하는 사설 클라우드, 베어메탈 등이 있습니다.

앞선 정의만 보면 동적 인프라는 클라우드와 굉장히 유사하지만, **키프 모리스**는 동적 인프라가 클라우드보다 범위가 더 넓다고 합니다.

📣 *동적 인프라를 소개하며 알려드리고 싶은 문장이 있습니다.* 

**클라우드로의 전환**이라는 의미에 대해 많은 사람들이 여러 측면에서 설명을 하지만,
저는 HashiCorp의 [Unlocking the Cloud Operating Model: Provisioning](https://www.hashicorp.com/resources/unlocking-the-cloud-operating-model-provisioning)
백서에 소개된 다음 표현에 참 공감이 갑니다.

> 클라우드로의 전환의 본질적인 의미는 "정적" 인프라에서 "동적" 인프라로의 전환입니다.
> 
> The essential implications of the transition to the cloud is the shift from “static” infrastructure to “dynamic” infrastructure

<br>

## 동적 인프라 플랫폼 요구 사항

동적 인프라 플랫폼은 다음과 같은 특성이 있습니다.

- Programmable
- On-Demand
- Self-Service

### 💻 Programmable

동적 인프라 플랫폼은 프로그래밍을 쉽게 할 수 있어야 합니다. 유저 인터페이스 외에도 스크립트, CLI와 같은 도구들과도 상호 작용 할 수 있도록 프로그래밍 API가 필요합니다.
아래 각 플랫폼 별 SDK를 사용하면 클라우드 내 자원을 생성하고 관리하는 코드를 작성할 수 있습니다.

- [AWS SDK](https://aws.amazon.com/tools/)
- [Azure SDK](https://azure.microsoft.com/en-us/downloads/)  
- [GCP SDK](https://cloud.google.com/sdk/)
- [Openstack SDK](https://docs.openstack.org/openstacksdk/latest/)


### ⏰ On-Demand

동적 인프라 플랫폼에서 자원을 즉시 생성하고 삭제하는 기능은 필수입니다.
또한 전통적인 인프라의 과금 정책이 일정 기간 동안의 계약을 기반으로 한다면, 동적 인프라에서는 시간당 과금 체계를 지원합니다. 


🎊 *드디어 대망의 셀프 서비스가 처음 소개 됩니다!*


### 🏃🏻 Self-Service

셀프서비스는 온디맨드 요구 사항을 좀 더 발전시킨 개념입니다. 전통적인 방법에서 인프라를 요구하기 위해서는 세부 요청 양식, 설계 및 명세 문서, 구현 계획 수립 등을 필요로 했습니다.
셀프서비스는 전통적인 방법에서 더 진화하여 **필요한 인프라를 즉시 프로비저닝하고 쉽게 수정할 수 있는 자동화된 절차**를 의미합니다.

🛠 *셀프 서비스는 인프라 템플릿을 운용할 수 있는 IaC 도구를 기반으로 구현합니다.* 

예를 들어 개발자가 로드밸런서로 ALB를 사용하고 있다가 NLB로 바꾸고 싶은 경우, 다음과 같이 정의된 인프라 코드를 재배포 하면 됩니다.

```terraform
resource "aws_lb" "test" {
  name               = "test-lb-tf"
  internal           = false
  
  # load_balancer_type = "application"
  load_balancer_type = "network"
  
  # Leave out other config ...
}
```

<br>

**HashiCorp**가 정의한 [Self-Service Infrastructure](https://www.hashicorp.com/products/terraform/self-service-infrastructure) 게시물을 보면,
Self-Service Infra가 어떤 의미인지 더 쉽게 다가옵니다. *(첨부된 링크를 통해 셀프서비스의 장점을 꼭 한번 읽어보세요!)*

#### 기존 방법

개발자는 인프라 담당자가 인프라를 할당할 때까지 기다려야 합니다.

<img src="https://www.hashicorp.com/img/products/terraform/use-cases/self-service-infrastructure/self-service-challenge.png" width="600" height="300">

#### 셀프 서비스 적용

작성된 인프라 템플릿을 활용해 온디맨드로 프로비저닝 할 수 있습니다.

<img src="https://www.hashicorp.com/img/products/terraform/use-cases/self-service-infrastructure/self-service-solution.png" width="600" height="300">

<br>

## 글을 마치며
저는 테라폼을 공부한 이후, 간단한 웹서비스를 운영하는 토이프로젝트를 진행할 때 다음과 같은 인프라 환경을 자주 사용합니다.

<img src="https://github.com/GSNextLevel/neoTerraform/blob/main/image/webserver_cluster.png?raw=true" width="500" height="550">

미리 코드로 정의한 인프라 덕분에 개발에만 집중할 수 있는 환경과 생산성 향상을 경험했습니다.
또한 테라폼 모듈 덕분에 제가 원하는 대로 인프라 스펙을 변경하고 각종 클라우드 서비스 추가 혹은 제거가 가능했습니다.

셀프서비스 인프라가 기존 승인 체계(*리소스 요청 ➡️ 담당자 승인*)를 부정하는 것은 아니라고 생각합니다.
기존 체계가 갖고 있는 보안적 이점을 포함한 장점들을 유지하며, 유연하고 신속하게 인프라를 운용하는 Self-service 환경이 불러올 장점을 고민해 봐야겠습니다.

마지막으로, DevOps 문화가 정착해가며 **CI/CD** 를 통해 지속적 배포가 가능해지며 더 잦은 서비스 출시가 가능해졌습니다.
또한, **IaC**를 통해 Immutable Infra를 추구하며 인프라의 일관성과 안정성을 보장하게 되었습니다.
앞선 두 개의 개념에 더해 **Self-service**를 추구한다면 조직의 민첩성과 생산성 향상에 도움이 될 것이라고 생각합니다.

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해주세요! 😃

---
