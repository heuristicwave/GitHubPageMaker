---
layout: post
current: post
navigation: True
title: Using the awslogs log driver in ECS(Fargate)
date: 2022-03-01 00:00:00
tags: [aws, devops]
class: post-template
subclass: 'post tag-aws'
author: HeuristicWave
---
ECS Task의 컨테이너가 생산하는 로그들은 CloudWatch를 활용하여 수집할 수 있습니다.
Cloudwatch Logs를 운영하며 로그 적재가 제대로 되지 않거나, Timestamp가 일치하지 않거나, 지나친 지연 시간이 발생하거나, 알아보기 어려운 형태로 로그가 쌓인다면 
아래 요소들을 고민해 볼 수 있습니다.

## 📚 References

- [Using the awslogs log driver](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html)
- [Regular expression Wikipedia](https://en.wikipedia.org/wiki/Regular_expression)

## Fargate에서 필요한 awslogs 로그 드라이버

공식 문서에서는 `awslogs-region`, `awslogs-group` 만이 필요하다고 하지만, **Fargate**를 사용하는 경우 `awslogs-stream-prefix`이 추가적으로 필요합니다.
또한, 가시성을 확보하기 위해 CloudWatch Logs에 수집된 여러 줄의 로그를 하나의 메시지로 보기 위해서는 `awslogs-multiline-pattern`이 필수적으로 필요합니다.

### awslogs-multiline-pattern

공식 문서의 Note 부분을 보면 다음과 같은 메모를 확인할 수 있습니다. *(정말 공식 문서는 한 줄도 그냥 지나칠 수 없는 것 같습니다!)*

> Multiline logging performs regular expression parsing and matching of all log messages.
> This may have a negative impact on logging performance.

실제로 저는 정규 표현식을 간과하고 검증되지 않은 정규식들을 적용했다가 다음과 같은 `Negative Impact`를 만났습니다.

1. 로그가 수집되기까지의 지나친 지연 시간 발생 (10분 이상)
2. 지연시간으로 인한 Timestamp 불일치 (Ingestion time과 Event Timestamp의 과도한 오차)
3. 1, 2번 이유로 인한 로그 미수집

<br>

## Regular Expression Lab

지금부터 예시들을 통해, CW Logs를 운영하며 만날 수 있는 상황들을 체험해 보겠습니다.

> *샘플 로그를 복사하여 [RegExr](https://regexr.com)에서 match 여부를 테스트해 볼 수 있습니다.* 

### Case 1️⃣

`awslogs-multiline-pattern`의 **value**로 `^INFO`를 설정할 경우 3개의 Line 중 match 되는 라인은 몇 라인일까요?

```shell
INFO | (pkg/trace/info/stats.go:104 in LogStats)                # Line 1
INFO | (pkg/trace/info/stats.go:104 in LogStats)                # Line 2
12:15:10 UTC | INFO | (pkg/trace/info/stats.go:104 in LogStats) # Line 3
```
<details><summary markdown="span">🖍 정답 보기</summary>

> **INFO** \| (pkg/trace/info/stats.go:104 in LogStats)                # Line 1

^(caret) 은 전체 문자열의 시작 위치에만 일치하므로, Line 1 만이 match 됩니다.

</details>

### Case 2️⃣

다음은 `시:분:초`를 표현하는 정규 표현식 입니다. 해당 정규 표현식은 아래 3줄을 모두 Match 시킬 수 있을까요? <br>
`(0[1-9]|1[0-9]|2[0-4]):(0[1-9]|[1-5][0-9]):(0[1-9]|[1-5][0-9])`

```shell
07:36:35 | Which line will match? Line 1
08:00:01 | I am Line 2!
08:01:00 | I am Line 3!
```

<details><summary markdown="span">🖍 정답 보기</summary>

> **07:36:35** \| I was matched <br>
> 08:00:01 \| I was not matched! <br>
> 08:01:00 \| I was not matched! <br>

그렇다면 왜? 첫 번째 라인만이 매칭되었을까요? **분, 초**에 해당하는 표현식을 유심히 살펴보면 00분 00시는 매칭되지 않습니다.
때문에 각각 (분 : `(0[0-9]|[1-5][0-9])`, 초 : `(0[0-9]|[1-5][0-9])`)로 수정해야 위 3줄을 매칭 시킬 수 있습니다.

</details>

<br>

### Other Case

위 2가지 케이스만 준비된다면 모든 로그들을 제대로 분리하여 수집할 수 있을까요?

<details><summary markdown="span">🤔 생각해보기</summary>

- Flag가 `INFO` 형식이 아닌 `WARN`이 발생할 경우
- Timestamp로 매칭 작업을 하는데 한 줄에 1회 이상 Timestamp가 포함된 경우
  > ex) **08:00:01** \| It's **08:02:03** right now.
- Application Crash로 인한 예상치 못한 메시지가 포함될 경우
- awslogs 로그 드라이버 내의 우선순위

</details>

<br>

아마도 위에 기재한 것들 외에도 더 고려 할 것들이 많을 것 같습니다.
개발이나 알고리즘 문제를 풀 때와 마찬가지로 항상 예상치 못한 실패 지점을 예상하는 습관이 필요한 것 같습니다.

<br>

---

{% include ecs_troubleshooting.html %}

<br>