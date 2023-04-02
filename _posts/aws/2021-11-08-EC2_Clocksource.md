---
layout: post
current: post
cover: assets/built/images/background/clock.png
navigation: True
title: EC2 Clocksource
date: 2021-11-08 00:00:00
tags: [aws]
class: post-template
subclass: 'post tag-aws'
author: HeuristicWave
---

몰라도 되지만 알면 알수록 더 신비한 EC2 🙃

# Preview

이번 포스팅에서는 [AWS Well-Architected Labs - Performance Efficiency](https://www.wellarchitectedlabs.com/performance-efficiency/100_labs/100_clock_source_performance/ )에 개재된 **Calculating differences in clock source**
를 읽고 궁금증이 생겨 구글링을 하다 알게 된 사실들을 의식의 흐름대로 작성한 포스팅입니다.

<br>

## Performance Efficiency Summary

일단 Performance Efficiency에 나오는 실험 내용을 요약하자면 다음과 같습니다.

AWS의 5세대 가상머신 Nitro와 non-nitro 인스턴스 2개를 올리고 시간을 반환하는 테스트 코드를 돌려 성능 테스트를 진행합니다.
당연히 5세대 Nitro가 기존 세대보다 월등한 결과를 보여 주지만,
non-nitro 기반의 인스턴스에서 **'리눅스 클럭 소스를 교체하면 유의미한 성능 향상의 결과를 얻을 수 있다'** 라는 실험 결과를 보여줍니다.

> Nitro 기반 인스턴스의 default clocksource : kvm-clock(권장)
>
> Non-nitro 인스턴스의 default clocksource : xen
> 
> 실험에서 교체한 Non-nitro 인스턴스의 clocksource : tsc

마지막으로 첨부된 [How do I manage the clock source for EC2 instances running Linux?](https://aws.amazon.com/premiumsupport/knowledge-center/manage-ec2-linux-clock-source)
게시물에서 클럭 소스를 교체하는 방법(*xen에서 tsc로 교체*)을 소개하며 실험 내용을 마칩니다.

<br>

## 궁금한 건 못 참아 ❓

위에 소개한 Lab을 진행하다 보니 *'왜 tsc로 교체하여 성능 향상 효과를 얻을 수 있는지'* 알 수가 없었습니다.
궁금증을 해소하기 위해 구글링을 하다 보니 이해를 돕는 다음 3가지 자료를 찾을 수 있었습니다.

### ⏱ Timestamping

[Red Hat Reference Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/reference_guide/chap-timestamping )에서 어느 정도 제 가려운 부분을 긁어 주었던 포스팅이 있습니다.

기본적으로 멀티프로세서 시스템인 NUMA와 SMP 아키텍처에서는 여러 개의 clock source가 탑재되어 있습니다.
멀티프로세서 기반의 EC2 인스턴스에서도 아래 명령어로 사용 가능한 clocksource를 확인하면 다음과 같은 결과를 확인할 수 있습니다.
```shell
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
xen tsc hpet acpi_pm
```

Red Hat의 실험 결과에 따르면 `tsc > hpet > acpi_pm` 순으로 오버헤드가 적은데,
`tsc`는 **register**에서 `hpet`은 **memory area**에서 읽기 때문에 수십만 개의 타임스탬프를 지정할 때 상당한 성능 이점을 제공한다고 합니다.

### ⚙️ Heap Engineering Post

[Running a database on EC2? Your clock could be slowing you down](https://heap.io/blog/clocksource-aws-ec2-vdso )을 보면 더 정확한 분석이 있습니다.
내용이 어려워 저는 완벽하게 이해하지 못했지만, 읽어보시면 굉장히 좋은 자료인 것 같습니다.

Heap Engineering 해당 포스팅에서 **밀당**을 시도하는데...
*'tsc에서는 낮은 가능성으로 clock drift 현상이 있어 프로덕션에서는 수행하지 말라'* 고 했다가,
실제로는 `clock drfit`가 발생하지 않는다며 [AWS가 tsc를 권장했던 슬라이드 자료](https://www.slideshare.net/AmazonWebServices/cmp402-amazon-ec2-instances-deep-dive/24 )를 함께 보여줍니다.

*그냥 맘놓고 `kvm-clock`이 탑재된 인스턴스를 사용하는게 좋을 것 같습니다.*

### 🎥 Tudum~ 또! Netflix

클라우드를 공부하다 보면 Netflix 가 클라우드에 지대한 영향을 끼친 것 같다고 느낄 때가 많은데, 이번에도 그랬습니다.
AWS re:Invent 2014에서 Netflix의 [Senior Performance Architect, Brendan Gregg의 발표 자료](https://www.slideshare.net/brendangregg/performance-tuning-ec2-instances/42 )를 보면
**xen에서 tsc로 교체**하여 **CPU 사용량은 30%, 평균 앱 레이턴시는 43%가 줄었다**고 합니다.

<br>

## Result

이번에도 구글링으로 딴짓을 하다 보니 많은 사실들을 알게 되었습니다. 사실 [Current generation instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#AvailableInstanceTypes )를
사용하면 대부분 위에서 언급한 최적화는 T2 시리즈, Gravition 계열을 제외한 대부분의 인스턴스에서는 기본적으로 적용되어 있습니다.
그래서 포스팅의 첫 포문을 '몰라도 되지만 ~'이라 지었습니다.

clocksource와는 별도로 이번 포스팅을 준비하다 거의 주말 하루를 소비했는데,
비교적 최근의 인스턴스가 과거 인스턴스들과 어떻게 다른지(Hypervisor, Jumbo Frame 등등)를 알 수 있었습니다.
새롭게 알게 된 사실들 역시 그냥 Nitro 기반의 Amazon Linux 2를 사용하면, 운영하는데 몰라도 지장 없이 최고의 성능을 보장받는 것 같습니다.
아직 알음알음 아는 지식이라 포스팅하기 어렵지만, 훗날 더 정확히 알게 되면 성능과 관련된 다른 튜닝 요소들도 적어보겠습니다. 

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해주세요! 😃

<br>

### 📚 포스팅과 직접적인 연관도는 떨어지지만 함께 보면 좋은 자료

- [AWS EC2 Virtualization 2017: Introducing Nitro](https://www.brendangregg.com/blog/2017-11-29/aws-ec2-virtualization-2017.html)
- [Linux AMI virtualization types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/virtualization_types.html)
- [Reinventing virtualization with the AWS Nitro System](https://www.allthingsdistributed.com/2020/09/reinventing-virtualization-with-nitro.html)

---

<br>