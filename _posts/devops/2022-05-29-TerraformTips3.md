---
layout: post
current: post
cover:  assets/built/images/background/terraform.png
navigation: True
title: Terraform Tips 3 - Refresh & Replace
date: 2022-05-29 00:00:00
tags: [devops, terraform]
class: post-template
subclass: 'post tag-devops'
author: HeuristicWave
---

Terraform 더 익숙하게 3 - Refresh & Replace


## Intro

IaC(Infrastructure as Code)를 운용하며 중요하게 생각하는 포인트 중 하나는, **코드로 정의한 형상**과 **실제 인프라의 형상**을 동일하게 유지하는 것 입니다.
Terraform에서는 **Configuration Drift**(정의한 형상과 달라지는 경우)를 방지하기 위해 다양한 명령어를 제공합니다.
이번 포스팅에서는 형상을 유지하는 다양한 기법 중 `Refresh`와 `Replace`에 대하여 알아보겠습니다.  

<br>

## Refresh

문서에는 다음과 같이 기재되어 있지만, 처음 접한다면 무엇을 말하는지 쉽게 와닿지 않습니다.

> The `terraform refresh` command reads the current settings from all managed remote objects and updates the Terraform state to match. <br>
> `terraform refresh` 명령어는 원격 객체의 현재 상태를 읽어 Terraform state와 일치시킵니다.

클라우드 환경에서 클러스터를 운용하면 인스턴스의 Scale이 변화함에 따라 인스턴스 ID 값도 변합니다.
이 경우 코드로 정의한 상태는 프로비저닝 당시 시점을 기억하지만, 실제 인프라의 현상은 최신 인스턴스의 상태를 가지고 있으므로 Drift가 발생합니다.

위와 같은 상황에서 `refresh` 명령어를 사용하면 `terraform.tfstate`의 값이 현재 인프라의 상태와 일치하게 됩니다.

하지만, 위 명령어는 deprecate 되었습니다. 왜냐하면 관리자가 무엇이 변경되는지 알지 못하고 `tfstate`가 최신화 되기 때문입니다.

<br>

## Outro



지금까지 테라폼 더 익숙하게 Refresh & Replace 편을 읽어주셔서 감사합니다! 잘못된 내용은 지적해 주세요! 😃

<br>

---

{% include terraform_tips.html %}

<br>