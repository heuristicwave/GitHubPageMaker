---
layout: post
current: post
cover:  assets/built/images/background/ecs.png
navigation: True
title: ECS on EC2 Walkthrough
date: 2022-07-03 00:00:00
tags: [aws]
class: post-template
subclass: 'post tag-aws'
author: HeuristicWave
---
EC2 기반의 ECS를 다루기 위한 사소한 지식들 톱아보기!

# Intro

당장 Amazon Elastic Container Service(이하 ECS)를 운영해야 하지만, 공식 문서를 다 읽기에는 벅차고 중요한 운영 포인트들을 빠르게 학습하기 위해 아래와 같은 요소들을 다룹니다.
*ECS 마스터*가 될 수 있는 모든 것을 다루는 것은 아니지만, 최소한의 고민해 볼 만한 지점들을 다뤄보았습니다.

1. Amazon ECS Container Agent
2. ECS 리소스 할당
3. ECS Scaling
4. ECS 서비스 구성

<br>

## 🕋 ECS Container Agent

EC2 기반의 ECS를 운용하기 위해서는 [ECS 최적화 AMI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html )를 사용해야 합니다.
일반 EC2 인스턴스로도 ECS를 운용할 수 있지만, ECS 최적화 AMI를 사용하는 것이 관리와 운용 측면에서 유리합니다.

> 예를 들어 Docker를 운영하다 보면, 미사용 상태인 컨테이너 이미지가 쌓여나가 `prune` 명령어를 통해 이미지를 정리해 주어야 합니다.
> 이런 상황에서 컨테이너 에이전트는 다양한 [자동화된 이미지 정리](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/automated_image_cleanup.html#automated_image_cleanup_parameters ) 옵션을 제공합니다.

ECS-optimized AMI에는 [Amazon ECS Container Agent](https://github.com/aws/amazon-ecs-agent )가 기본적으로 포함되어 있습니다.
ECS 인스턴스를 부트스트랩 하는 단계에서 EC2의 user data를 사용해 `/etc/ecs/ecs.config`에 configuration parameters을 전달합니다.

아래와 같이 환경 변수를 지정하지 않아도 default 값이 지정되어 있어 운용상의 큰 문제는 없지만,
[공식 문서](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-agent-config.html )를 확인해 필요한 configuration 들을 파악해야 합니다.

```shell
#!/bin/bash
cat <<'EOF' >> /etc/ecs/ecs.config
ECS_CLUSTER=MyCluster
ECS_LOGLEVEL=debug
EOF
```

다양한 configuration 중, 운영 환경(GPU, SPOT, 서비스 등)에 따라 필요한 Configuration 값들이 다르겠지만 일반적으로 `ECS_RESERVED_MEMORY`는 고려하여 지정하는 것이 좋습니다.

> 💡 인스턴스의 모든 메모리를 테스크에 배정한다면, 테스크와 중요한 시스템 프로세스 사이에서 메모리 경합이 발생할 수 있습니다.
> 이를 예방하기 위해 `ECS_RESERVED_MEMORY` configuration을 사용해 메모리를 예약함으로써 풀에서 할당 가능한 메모리를 제외할 수 있습니다.
> 필요로 하는 최소한의 요구 메모리가 정의되어 있지는 않지만, 저는 문서의 예시처럼 256MiB로 사용하고 있습니다. 

<br>

## 🎛 ECS Resource

ECS는 3가지 범주로 리소스를 설정할 수 있습니다.

🥇 **Container instance** : ECS 클러스터를 이루고 있는 EC2 인스턴스의 유형 <br>
🥈 **Task Size** : Task Definitions을 통해 정의하는 Size <br>
🥉 **Container Size** : Task Definitions의 Container 정의 부분에서 정의하는 Size <br>

> 💡 Container Size의 리소스 할당 파라미터는 각각 다음과 같은 `docker run` 명령어 옵션에 매핑됩니다. <br>
> **cpu** : `--cpu-shares`, **memory** : `--memory`(hard) / `--memory-reservation`(soft)
> <br><br>
> 💡 ECS on EC2에서는 Task Size와 Container Size 방식 중 선택 가능하지만, Fargate 방식에서 Task Size는 필수입니다.

🥇로 갈수록 더 상위 개념이며, Task Definitions에서는 🥈, 🥉을 활용해 리소스를 제어합니다.
세밀한 제어를 하고 싶다면 🥈, 🥉을 모두 사용하여 제어할 수 있으나,
🥉 사용해야 하는 특별한 이유가 없다면 🥈만을 사용해 리소스를 제어하는 것이 용이합니다. 

### Actual available memory

16GiB의 인스턴스를 프로비저닝해도 **실제 사용 가능한 메모리(15318 MiB)**는 더 적습니다.

|Amazon EC2|인스턴스|`free -m`|Registered|
|---|---|---|---|
|c5.2xlarge|16384 MiB|15574 MiB|15318 MiB|

EC2 인스턴스와 `free -m` 명령어로 확인한 차이

*ECS 플랫폼 메모리 오버헤드와 OS 커널이 차지하는 메모리로 인해 차이가 발생합니다.*

`free -m` 명령어로 확인한 메모리와 컨테이너 인스턴스(registered)의 차이

*컨테이너 에이전트를 설정할 때, `ECS_RESERVED_MEMORY=256`으로 설정한 만큼 차이가 발생합니다.* 

<br>

## 🏘 ECS Scaling

ECS의 Scaling 방법은 2가지로 정의할 수 있습니다. 해당 스케일링 기법은 선택 사항이 아니라, 2가지 모두를 고려해 적용해야 합니다. 

1. Service에 의하여 Task가 Scaling 되는 Horizontal Autoscaling
2. CapacityProvider에 의하여 컨테이너 인스턴스가 Scaling 되는 Cluster Autoscaling

### Service auto scaling

서비스 스케일링은 또다시 [Target tracking](https://docs.aws.amazon.com/AmazonECS/latest/userguide/service-autoscaling-targettracking.html )과 [Step](https://docs.aws.amazon.com/AmazonECS/latest/userguide/service-autoscaling-stepscaling.html) scaling으로 나뉩니다.

Target tracking은 **CPU/Memory 사용률** 및 **ALB 요청 횟수**를 기반의 정책이 있으며, Step 방식은 **Alarm을 활용한 Custom** 정책들을 작성할 수 있습니다.

> 💡 Step scaling policies의 문서 첫 문장은 다음과 같습니다. <br>
> Although Amazon ECS service auto scaling supports using Application Auto Scaling step scaling policies, we recommend using target tracking scaling policies instead. <br>

Step scaling은 조금 더 공격적인 정책이 필요할 때 대안으로 사용하고, Service auto scaling은 복수의 조정 정책을 동시에 활용할 수 있으므로 하나의 정책에 의존하기보다는 복수의 정책을 사용해 보다 세밀한 Scaling 정책을 만드는 게 어떨까요? 🧐

### Cluster auto scaling

Service auto scaling에 컨테이너 인스턴스 내 자원을 다 할당했을 경우, CapacityProvider EC2 Auto Scaling을 활용해 클러스터 자원을 확보합니다.

[AWS 블로그](https://aws.amazon.com/blogs/containers/deep-dive-on-amazon-ecs-cluster-auto-scaling/ )에 자세한 작동원리가 나오므로 꼭 한번 읽어보시기 바랍니다.
해당 내용을 요약하자면 **Capacity Provider**를 Cluster 인프라를 관리합니다.
이를 위해 **CapacityProviderReservation 지표**가 존재하고 사전에 설정한 **Target capacity %**(1~100사이의 값)에 맞춰 EC2 AutoScalingGroup(ASG)에 Trigger가 작동합니다.

`CapacityProviderReservation(%) = M(현재 인스턴스+추가 요청 인스턴스)/N(현재 인스턴스) * 100(%)`

만약 Target capacity을 100으로 지정했을 때, 현재 Cluster의 Node 개수가 3이고 CapacityProviderReservation가 200이라면
CapacityProviderReservation를 목표치(Target capacity)인 100에 맞추기 위해 3개의 EC2를 Scale-out 시킵니다.

> ⚠️ To create an empty Auto Scaling group, set the desired count to zero. <br> 
> Capacity Provider는 **Desired** 값이 **0**인 ASG와 연결되어야 합니다. 또한 CapacityProvider에 의하여 관리형 조정을 enable 한 상태에서,
> ASG를 **수동**으로 수정한다면 CapacityProviderReservation 계산에 영향을 미칠 수 있으므로 **지양**해야 합니다. 
> <br><br>
> 🚫️ DO NOT EDIT OR DELETE <br>
> 해당 메시지는 Service & Cluster Scaling이 CloudWatch에 자동으로 생성한 TargetTracking의 주석 내용입니다.
> 관리형 정책의 기능을 활용 시, 조금 더 세밀한 Scaling 필요하다면 알람 Trigger의 빈도가 아닌 다른 방안을 고민하도록 합니다! 

<br>

## 🧮 ECS Service Configuration

### Deployment Configuration

ECS 서비스에서 [배포와 관련된 설정](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_DeploymentConfiguration.html )을 보면, `maximumPercent`와 `minimumHealthyPercent`가 있습니다.
제게 비슷하면서도 헷갈리기 쉬운 2개의 설정값 개념은 손에 잡힐듯하면서도 쉽게 잡히지 않는 것 같습니다.😂

*해당 개념이 조금 더 와닿을 수 있도록, 공식 문서를 읽어보시고 아래 상황이 어떨지 예상해 보세요!*

🧑🏻‍💻는 ECS `maximumPercent`가 200%, `minimumHealthyPercent`가 100%로 설정되어 있으며, Rolling update 방식을 사용하고 있다.
현재 ver01을 운영하는 🧑🏻‍💻는 실수로 오류를 포함한 ver02를 배포했다. 기존 Task 4개일 때, 어떤 상황이 벌어지는가?

<details><summary markdown="span">🖍 정답 보기</summary>

`minimumHealthyPercent`가 100%이기 때문에 ver02에 프로비저닝되고 정상 상태로 확인될 때까지 ver01은 중단되지 않습니다.
ver02는 **running** 상태로 진입하지 못해 **provisioning-pending-stopped** 단계를 반복합니다.
이때 `maximumPercent`가 200%임에 따라 ver02 Task는 4개와 ver01 Task 4개(합, 최대 8개)의 Task가 동시에 올라올 수 있습니다. 

</details>

### Task Placement

ECS의 서비스가 컨테이너 인스턴스에 Task를 배치하는 전략은 아래와 같이 3가지 분류됩니다.

- `binpack`: CPU 또는 메모리를 최소화하기 위해 유휴 자원을 고려한 배치
- `random` : 무작위 배치
- `spread` : ami-id, availability-zone, instance-type, os-type 등을 고려한 배치

위 3가지 전략은 단독 혹은 복수로 선택되어 사용될 수 있으며, 가용성을 확보하기 위해 AZ를 고려하여 `AZ + binpack`, `AZ + spread`와 같이 사용되기도 합니다.

*쿠버네티스를 공부해 보신 분이라면, 해당 전략은 마치 k8s의 nodeSelector와 비슷하게 동작합니다.* 

> 🔊 CapacityProviderReservation에 영향을 미치는 배포 설정과 배치 전략 <br>
> ECS 서비스 설정에서 언급한 배포 설정과 배치 전략은 가용성 문제와 직결되고 이는 비용 문제로도 이어집니다.
> 앞서 언급 한 CapacityProviderReservation 계산에 활용되는 M 값에 배포 설정과 배치 전략이 영향을 미친다는 점을 유의하세요! 

<br>

## Outro

이번 포스팅을 통해 ECS와 관련된 내용을 정리하다 보니, AWS 내의 다른 오케스트레이션 서비스인 EKS의 운용 전략과 무척이나 비슷하다는 느낌을 지울 수 없었습니다.
쿠버네티스의 HPA 최적화, Pod 배치 및 리소스 할당 전략과 같은 포인트들은 운영을 하며 지속적으로 관리하는 관리 대상인 만큼,
위에 언급된 ECS의 운영 포인트들도 **서비스를 배포한 이후에도 지속적으로 관심**을 가져야 하는 포인트임을 강조하며 마칩니다.

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해 주세요! 😃

<br>

---

<br>