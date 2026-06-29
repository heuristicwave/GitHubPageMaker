---
layout: post
current: post
cover: assets/built/images/background/s3-annotations-ai-sre-context-layer.svg
navigation: True
title: "Amazon S3 Annotations 3편: AI SRE를 위한 Observability Context Layer 검증하기"
display_title: "Amazon S3 Annotations 3편:<br>AI SRE를 위한 Observability Context Layer 검증하기"
date: 2026-06-29 00:00:00
tags: [aws, genai, lab]
class: post-template
subclass: "post tag-ai"
author: HeuristicWave
---

[1편](/S3-OKF-Annotations.html)에서는 Markdown 문서를 S3 Object로 올리고, frontmatter, links, summary를 각각 S3 Annotations로 분리해보았습니다.
[2편](/S3-Annotations-2.html)에서는 S3 Annotations를 특정 문서 포맷의 부가 정보가 아니라, S3 객체 전반에 의미 계층을 붙이는 context layer로 보는 관점을 정리했습니다.

2편의 마지막에는 다음 질문을 남겼습니다.

> S3 Annotations 기반 context layer가 실제 운영 문제에서도 유의미할까?

이번 글에서는 이 질문을 AI SRE 예시로 검증해보겠습니다.
비교를 위해 Agent가 볼 수 있는 정보를 단계별로 나눴습니다.
application log만 보는 경우, 같은 log 범위에 배포/설정/metric/runbook 맥락을 더하는 경우, 그리고 S3 Annotations로 service, deployment, telemetry, runbook 사이의 관계를 따라가게 하는 경우를 분리했습니다.

결론부터 말하면, **Annotations 기반 Context Layer는 로그를 대신 읽어주는 기능이 아니라, Agent가 먼저 볼 객체와 운영 맥락을 좁혀주는 구조**에 가까웠습니다.
raw log에 운영 맥락을 그대로 덧붙이는 방식보다 적은 token으로 더 나은 RCA 결과를 냈고, downstream 호출 실패 대상과 실제 원인 service를 구분하는 데도 도움이 됐습니다.

<br>

## <a href="#question">🤔 검증할 질문</a><a id="question"></a>

장애 상황에서 사용자는 보통 이렇게 묻습니다.

```text
payment-service의 5xx가 갑자기 늘어난 이유를 찾아줘.
```

이 질문에 답하려면 로그를 읽어야 합니다.
하지만 로그만 보면 "내가 호출한 다음 서비스가 느리다"는 증상을 실제 원인으로 착각하기 쉽습니다.
여기서 downstream은 `payment-service`가 호출하는 `fraud-score-api`나 `ledger-service` 같은 뒤쪽 서비스를 뜻합니다.
예를 들어 `payment-service` 로그에 `downstream=fraud-score-api`, `failure_reason=fraud_timeout`이 반복되면 `fraud-score-api`가 문제처럼 보입니다.
하지만 실제로는 `payment-service` 배포에서 timeout 값을 너무 짧게 바꾼 것이 원인일 수도 있습니다.

이 글에서는 Annotations 기반 Context Layer를 다음 세 질문으로 나눠 보겠습니다.

1. Agent가 읽어야 할 로그 범위를 줄이는가?
2. 배포/설정/runbook 같은 운영 맥락을 더 작게 전달할 수 있는가?
3. 호출 실패 대상과 실제 원인 service를 구분하는 데 도움이 되는가?

<br>

## <a href="#setup">🧱 실험 환경</a><a id="setup"></a>

실험은 세 개의 service와 다섯 개의 장애 시나리오로 구성했습니다.
Agent는 같은 질문을 받고, 각 비교군에 허용된 도구와 context만 사용해 root cause를 판단합니다.

서비스는 세 개입니다.

| Service | 역할 |
| --- | --- |
| `payment-service` | 결제 요청을 받고 fraud check와 ledger persistence를 호출 |
| `fraud-score-api` | fraud score를 계산하고 allow/deny decision 반환 |
| `ledger-service` | 결제 결과를 DynamoDB에 저장 |

장애 시나리오는 다음과 같습니다.

| Scenario | 실제 원인 | 의도한 장애 |
| --- | --- | --- |
| `payment-timeout` | `payment-service` | fraud timeout/fail-closed 설정 변경 |
| `fraud-latency` | `fraud-score-api` | fraud-score-api latency 증가 |
| `ledger-persistence` | `ledger-service` | ledger-service persistence 실패 |
| `payment-ledger-timeout` | `payment-service` | ledger timeout/fail-closed 설정 변경 |
| `fraud-error-rate` | `fraud-score-api` | fraud model error rate 증가 |

<br>

## <a href="#matrix">🧪 비교군을 나누기</a><a id="matrix"></a>

핵심은 비교군을 분리한 것입니다.

| ID | 비교군 | 제공 정보 | 목적 |
| --- | --- | --- | --- |
| A | 원본 로그 | 3개 service의<br>application log 원문 | 로그만 볼 때의<br>baseline |
| B | 원본 로그<br>+ 운영 맥락 | A와 같은 원본 로그<br>운영 맥락 원문 | 운영 맥락을<br>그대로 추가 |
| C | Annotation<br>Context Layer | Annotation 관계 그래프<br>raw telemetry row | 관계 기반 context와<br>원본 telemetry |
| D | Annotation<br>Signal Summary | C와 같은 telemetry 범위<br>signal summary | 선택된 telemetry를<br>작게 전달 |

이렇게 나눈 이유는 단순합니다.
Annotations 기반 Context Layer를 보려면 “운영 맥락을 더 봐서 좋아진 것”과 “그 운영 맥락을 어떤 구조로 전달했는가”를 분리해야 합니다.

특히 B는 A와 같은 3개 service 로그를 모두 읽도록 맞췄습니다.
그 위에 deployment 기록, task definition의 config diff, metric summary, runbook, service topology를 추가로 읽습니다.
그래야 원본 로그 + 운영 맥락 비교군이 원본 로그 비교군보다 적은 로그를 읽어서 유리해지는 상황을 피할 수 있습니다.

반면 C는 Annotation 관계 그래프가 가리키는 telemetry object를 따라갑니다.
다만 C는 로그 row를 줄이지 않습니다.
Annotation graph로 incident와 연결된 service, deployment, telemetry object를 찾은 뒤, 그 telemetry object의 raw rows를 그대로 읽습니다.

D는 C와 같은 telemetry 범위를 읽되, raw rows를 그대로 Agent에게 넘기지 않고 `http_status`, `latency`, `downstream_call`, `error_count` 같은 signal summary로 줄입니다.
따라서 D는 “Annotation이 맞는지”를 보는 주 비교군이라기보다, Annotation 기반 Context Layer를 운영할 때 token을 줄이는 방법을 확인하기 위한 Tip 성격의 비교군입니다.

<br>

## <a href="#architecture">🧭 로그와 Annotation은 어디에 있는가</a><a id="architecture"></a>

이 실험에서 로그의 원천은 application이 남기는 CloudWatch Logs입니다.
`payment-service`, `fraud-score-api`, `ledger-service`가 각각 자신의 log group에 request, status code, latency, downstream 호출 결과를 남깁니다.
그중 incident window에 해당하는 로그와 metric을 S3 telemetry object로 export했다고 가정했습니다.

S3에는 다음과 같은 객체가 놓입니다.

| 객체 종류 | 예시 | 역할 |
| --- | --- | --- |
| service object | `services/payment-service` | 서비스 자체를 표현 |
| deployment object | `deployments/payment-service/...` | 배포 버전과 config diff 표현 |
| telemetry object | `telemetry/payment-service/...` | incident window의 log/metric 묶음 |
| runbook object | `runbooks/payment-service.md` | 조사 절차와 확인 항목 |
| incident object | `incidents/payment-ledger-timeout` | 특정 장애 window와 affected service |

S3 Annotations는 이 객체들 옆에 붙는 context입니다.
예를 들어 incident object에는 affected service와 incident window를 붙이고, deployment object에는 `LEDGER_TIMEOUT_MS: 800 -> 150`, `LEDGER_FAIL_CLOSED: false -> true` 같은 config diff를 붙입니다.
telemetry object에는 `http_status`, `latency`, `downstream_call`, `error_count` 같은 일반적인 `telemetry.signal`을 붙입니다.
그리고 `context.relations`로 incident, deployment, telemetry, runbook 사이의 관계를 연결합니다.

```text
incident
  -> affected_service: payment-service
  -> related_deployment: payment-service deployment
  -> related_telemetry: payment telemetry, fraud telemetry
  -> related_runbook: payment-service runbook
```

이 구조의 목적은 정답을 annotation에 써두는 것이 아닙니다.
Agent가 "원본 로그를 해석하기 전에 어떤 객체와 운영 맥락을 함께 봐야 하는지"를 알려주는 것입니다.

<br>

## <a href="#agent-tools">🧰 Agent별 Tool 구조</a><a id="agent-tools"></a>

네 조건은 같은 질문을 받지만 읽을 수 있는 정보의 범위와 전달 방식이 다릅니다.
여기서 A와 B의 로그 조회는 같은 원본 로그를 봅니다.
B는 A와 같은 `query_service_logs`를 service별로 호출하고, 그 뒤에 운영 맥락 tool을 추가로 호출합니다.
그래야 B가 A보다 적은 로그를 읽어서 유리해지는 상황을 피할 수 있습니다.

| 비교군 | 호출 가능한 Tool | 읽을 수 있는 정보 |
| --- | --- | --- |
| A. 원본 로그 | `query_service_logs` | service별 application log |
| B. 원본 로그 + 운영 맥락 | `query_service_logs`<br>`list_recent_deployments`<br>`get_task_definition_diff`<br>`get_metric_summary`<br>`get_runbook`<br>`get_service_topology` | A와 같은 원본 로그<br>배포 이력/config diff<br>metric/runbook/topology |
| C. Annotation Context Layer | `search_annotation_incidents`<br>`get_annotation_bundle`<br>`get_raw_telemetry_rows` | incident 관련 Annotation 관계 그래프<br>연결된 객체 bundle<br>raw telemetry row |
| D. Annotation Signal Summary | `search_annotation_incidents`<br>`get_annotation_bundle`<br>`get_telemetry_signal_summary` | C와 같은 Annotation 관계 그래프<br>signal summary |

즉 A는 로그만 봅니다.
B는 로그에 운영 도구 결과를 그대로 더합니다.
C는 먼저 Annotation 관계 그래프를 조회하고, incident와 관련된 annotation bundle과 service 범위를 얻습니다.
D는 C와 같은 범위를 보되, raw telemetry row를 signal summary로 줄여서 전달합니다.
실험 코드의 실제 함수명은 `get_compact_annotation_bundle`이었지만, 여기서 compact는 telemetry를 압축했다는 뜻이 아니라 incident와 연결된 annotation object 묶음만 가져온다는 구현명입니다.

<br>

## <a href="#scenario-flow">🔬 실제 시나리오 입력과 흐름</a><a id="scenario-flow"></a>

예를 들어 `payment-ledger-timeout` 시나리오에서 Agent에게 들어간 공통 입력은 다음과 같습니다.

```text
질문: payment-service의 5xx가 갑자기 늘어난 이유를 찾아줘
affected_service: payment-service
candidate_services: payment-service, fraud-score-api, ledger-service
incident_window: 2026-06-28T14:20:36Z ~ 2026-06-28T14:24:01Z
```

원본 로그 Agent는 세 service의 로그를 직접 조회합니다.

```text
1. query_service_logs(payment-service)   -> 22 rows
2. query_service_logs(fraud-score-api)   -> 22 rows
3. query_service_logs(ledger-service)    -> 0 rows
```

이 Agent는 `payment-service` 로그에 반복되는 `ledger-service` timeout을 보고 `ledger-service`를 1순위 원인으로 판단했습니다.

원본 로그 + 운영 맥락 Agent는 같은 로그를 읽고, 운영 맥락 도구를 추가로 호출합니다.

```text
1. query_service_logs(fraud-score-api)   -> 22 rows
2. query_service_logs(payment-service)   -> 22 rows
3. query_service_logs(ledger-service)    -> 0 rows
4. get_task_definition_diff(payment)     -> LEDGER_TIMEOUT_MS, LEDGER_FAIL_CLOSED
5. list_recent_deployments               -> 1 record
6. get_task_definition_diff(ledger)      -> no changed fields
7. get_runbook(payment-service)          -> 5 steps
8. get_service_topology                  -> 3 edges
9. get_metric_summary                    -> 10 rows
```

이 Agent는 `payment-service`의 `LEDGER_TIMEOUT_MS`가 800ms에서 150ms로 바뀐 사실을 봤습니다.
또한 `ledger-service` 쪽에는 대응되는 설정 변경이 없다는 것도 확인했습니다.
그래서 호출 실패 대상으로 찍힌 `ledger-service`가 아니라, timeout 설정을 바꾼 `payment-service`를 1순위 원인으로 판단했습니다.

Annotation Context Layer Agent는 먼저 Annotation 관계 그래프를 조회합니다.

```text
1. search_annotation_incidents(payment-service, window) -> 1 incident
2. get_annotation_bundle(incident_ref)                  -> 5 objects
3. get_raw_telemetry_rows(selected objects)             -> log 44 rows, metric 10 rows
```

이 흐름에서는 incident object가 payment deployment, telemetry object, runbook과 연결되어 있습니다.
따라서 Agent는 `ledger-service 호출이 실패했다`는 증상과 함께, 그 시점의 `payment-service` 배포 설정 변경을 같은 context 안에서 보게 됩니다.
그 결과 `payment-service`를 1순위 원인으로 판단했습니다.

여기서 중요한 점은 C가 로그 row를 줄이지 않았다는 것입니다.
이 시나리오에서 A, B, C는 모두 같은 44 log rows를 읽었습니다.
C의 실제 tool 호출은 다음과 같았습니다.

```text
get_raw_telemetry_rows(
  object_refs = ["obj-7", "obj-8"],
  services    = ["payment-service", "fraud-score-api", "ledger-service"],
  max_rows    = 120
)
```

`object_refs`는 Annotation graph가 고른 telemetry object를 뜻합니다.
이번 최종 실험에서는 이 object가 가리키는 raw telemetry rows를 fault-first prefilter 없이 그대로 읽게 했습니다.
따라서 C의 효과는 "로그를 덜 읽어서 좋아졌다"가 아니라, **같은 raw telemetry를 읽더라도 service, deployment, telemetry, runbook 관계를 먼저 구조화해서 본 효과**로 해석해야 합니다.

<br>

## <a href="#annotation-flow">🏷️ Annotation은 무엇을 알려주는가</a><a id="annotation-flow"></a>

이번 실험에서 S3 Annotations는 raw log를 대체하지 않습니다.
로그 본문은 CloudWatch Logs와 S3 telemetry object에 남아 있고, annotation은 그 object 옆에 붙은 context입니다.

```text
1. service가 CloudWatch Logs에 application log 기록
2. incident window의 로그와 metric을 S3 telemetry object로 export
3. service / deployment / telemetry / runbook / incident object 저장
4. 각 object 옆에 S3 Annotations 부여
5. Agent가 Annotation 관계 그래프를 따라 필요한 telemetry를 선택
```

여기서 중요한 점은 두 가지입니다.

첫째, `deploy.release`는 해석형 risk flag가 아니라 사실 정보 중심으로 다뤘습니다.
`fraud_timeout_changed` 같은 정답 힌트 대신, 변경된 설정값과 배포 맥락을 운영 맥락 원문이나 annotation bundle로 제공합니다.

둘째, `telemetry.signal`은 정답을 알려주는 label이 아니라 일반적인 signal family로 표현했습니다.
이번 실험에서 사용한 signal family는 다음 정도입니다.

```json
["http_status", "latency", "downstream_call", "error_count"]
```

즉 Agent에게 “이 장애의 원인은 payment-service다”를 알려주는 것이 아니라, “이 telemetry object에는 HTTP 상태, latency, downstream call, error count 관찰값이 있다” 정도의 색인을 주는 구조입니다.

<br>

## <a href="#result">📊 전체 결과</a><a id="result"></a>

전체 결과는 다음과 같습니다.
여기서 Accuracy는 1순위 root cause가 맞았는지를 뜻하고, Top-2는 Agent가 제시한 상위 2개 후보 안에 실제 원인이 포함됐는지를 뜻합니다.

| 비교군 | Accuracy | Top-2 | Total tokens | Logs read | Tool calls | 의미 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| 원본 로그 | 3/5 | 5/5 | 43,976 | 196 | 15 | raw log baseline |
| 원본 로그<br>+ 운영 맥락 | 5/5 | 5/5 | 100,084 | 196 | 42 | 같은 원본 로그<br>운영 맥락 추가 |
| Annotation<br>Context Layer | 5/5 | 5/5 | 75,090 | 196 | 15 | 관계 그래프 + raw telemetry<br>annotation object 30개 읽음 |
| Annotation<br>Signal Summary | 5/5 | 5/5 | 51,023 | 196 | 15 | C와 같은 telemetry 범위<br>signal summary로 전달 |

원본 로그만 본 Agent는 3/5였습니다.
`payment-timeout`, `payment-ledger-timeout`처럼 로그에 보이는 호출 실패 대상과 실제 원인 배포 service가 다른 케이스에서 오판했습니다.

원본 로그 + 운영 맥락 Agent는 5/5였습니다.
3개 service 로그를 모두 읽고 deployment/config/metric/runbook/topology까지 보면서 5/5를 맞췄습니다.
다만 token은 100,084로 가장 컸습니다.

Annotation Context Layer를 사용한 Agent는 5/5였습니다.
이 Agent는 raw telemetry row를 줄이지 않고 A/B와 같은 196 log rows를 읽었습니다.
그래도 Total tokens는 75,090으로, 원본 로그 + 운영 맥락의 100,084보다 약 25.0% 적었습니다.

Annotation Signal Summary Agent도 5/5였습니다.
C와 같은 telemetry 범위 196 log rows를 바탕으로 signal summary를 만들었고, Total tokens는 51,023이었습니다.
이는 C보다 약 32.1%, 원본 로그 + 운영 맥락보다 약 49.0% 적습니다.

<br>

## <a href="#three-answers">🧾 세 질문에 대한 답</a><a id="three-answers"></a>

첫째, Agent가 읽어야 할 로그 범위를 줄였는가?

이번 실험에서는 **줄였다고 말하지 않는 편이 맞습니다.**
원본 로그 Agent, 원본 로그 + 운영 맥락 Agent, Annotation Context Layer Agent는 모두 총 196 log rows를 읽었습니다.
C는 Annotation 관계 그래프로 telemetry object를 찾았지만, 그 object의 raw telemetry row는 선별 없이 그대로 읽었습니다.

**따라서 이번 결과의 핵심은 log row 수 감소가 아닙니다.**
핵심은 같은 raw telemetry를 읽더라도, Annotation graph가 incident, deployment, telemetry, runbook 관계를 먼저 구조화해 준다는 점입니다.

둘째, 운영 맥락을 더 작게 전달했는가?

원본 로그 + 운영 맥락 Agent는 원본 로그 Agent와 같은 196 log rows를 읽고, 그 위에 deployment/config/metric/runbook/topology를 추가로 읽었습니다.
그 결과 token은 크게 늘었습니다.

<div class="language-text highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span style="display:inline-block; min-width: 230px;">원본 로그:</span><span style="display:inline-block; width: 140px; text-align: right;">43,976 tokens</span>
<span style="display:inline-block; min-width: 230px;">원본 로그 + 운영 맥락:</span><span style="display:inline-block; width: 140px; text-align: right;">100,084 tokens</span>
<span style="display:inline-block; min-width: 230px;">증가율:</span><span style="display:inline-block; width: 140px; text-align: right;">약 127.6%</span></code></pre></div></div>

반면 Annotation Context Layer는 같은 raw telemetry rows를 읽되, 운영 맥락을 Annotation graph와 annotation bundle로 전달했습니다.

<div class="language-text highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span style="display:inline-block; min-width: 230px;">원본 로그 + 운영 맥락:</span><span style="display:inline-block; width: 140px; text-align: right;">100,084 tokens</span>
<span style="display:inline-block; min-width: 230px;">Annotation Context Layer:</span><span style="display:inline-block; width: 140px; text-align: right;">75,090 tokens</span>
<span style="display:inline-block; min-width: 230px;">감소율:</span><span style="display:inline-block; width: 140px; text-align: right;">약 25.0%</span>
<span style="display:inline-block; min-width: 230px;">Accuracy:</span><span style="display:inline-block; width: 140px; text-align: right;">5/5 -> 5/5</span></code></pre></div></div>

따라서 이 실험에서는 **raw log에 운영 변경 맥락을 그대로 붙이는 것보다, Annotations 기반 Context Layer로 관계를 구조화해 전달하는 방식이 같은 정확도를 더 작은 입력으로 만들었습니다.**

셋째, 호출 실패 대상과 실제 원인 service를 구분하는 데 도움이 됐는가?

도움이 됐습니다.
`payment-ledger-timeout`에서는 원본 로그 Agent가 `ledger-service`를 1순위로 잘못 골랐습니다.
로그에 보이는 것은 `payment-service -> ledger-service` 호출 timeout이었기 때문입니다.
반면 운영 맥락을 추가한 B와 Annotation Context Layer를 사용한 C/D는 `payment-service`를 원인으로 판단했습니다.

```text
원본 로그:                    ledger-service 오판
원본 로그 + 운영 맥락:        payment-service 정답
Annotation Context Layer:      payment-service 정답
Annotation Signal Summary:     payment-service 정답
```

즉 이 실험에서 가장 강하게 말할 수 있는 결론은 이것입니다.

> **Annotations 기반 Context Layer는 raw log를 대체하지 않는다. 대신 Agent가 먼저 연결해서 볼 service, deployment, telemetry object의 관계를 구조화해 주면서, 호출 실패 대상과 실제 원인 service를 구분하는 데 도움을 줄 수 있다.**

<br>

> **Tip: 선택된 로그도 그대로 넣으면 token이 낭비될 수 있습니다.**
>
> Agent D는 이 가정을 따로 확인하기 위해 둔 조건입니다.
> D는 C와 같은 Annotation graph를 사용하고, 같은 raw telemetry 범위 196 rows를 읽었습니다.
> 차이는 raw rows를 그대로 Agent에게 넘기지 않고 `http_status`, `latency`, `downstream_call`, `error_count` 같은 RCA용 signal summary로 줄여 전달했다는 점입니다.
> 기본 원리는 단순합니다.
> 196개 log row를 줄줄이 넣는 대신, 같은 패턴의 row를 service, status code, downstream 대상, error type, latency 구간 기준으로 묶습니다.
> 그리고 각 묶음마다 count, 대표 latency, 대표 error message, 예시 row 몇 개만 남깁니다.
> 예를 들어 `payment-service`의 `ledger-service timeout` row가 여러 번 반복된다면, Agent에게 모든 row를 보여주는 대신 “payment-service에서 ledger-service timeout이 N회 발생했고, latency는 이 범위였으며, 대표 에러는 이것이다”라는 형태로 전달합니다.
> 그 결과 C는 75,090 tokens, D는 51,023 tokens를 사용했습니다.
> 즉 같은 telemetry 범위에서도 signal summary를 쓰면 token을 약 32.1% 줄일 수 있었습니다.

<br>

## <a href="#downstream-trap">🔎 호출 실패 대상에 속는 케이스(downstream)</a><a id="downstream-trap"></a>

시나리오별 관찰은 다음과 같습니다.

| Scenario | 원본 로그 | 원본 로그 + 운영 맥락 | Annotation Context Layer | Annotation Signal Summary |
| --- | --- | --- | --- | --- |
| `payment-timeout` | `fraud-score-api` 오판 | 정답 | 정답 | 정답 |
| `fraud-latency` | 정답 | 정답 | 정답 | 정답 |
| `ledger-persistence` | 정답 | 정답 | 정답 | 정답 |
| `payment-ledger-timeout` | `ledger-service` 오판 | 정답 | 정답 | 정답 |
| `fraud-error-rate` | 정답 | 정답 | 정답 | 정답 |

흥미로운 케이스는 `payment-ledger-timeout`입니다.
raw log에는 `payment-service`가 `ledger-service`를 호출하다 timeout이 났다는 기록이 반복됩니다.
이 로그만 강하게 보면 “ledger-service가 느려서 장애가 났다”고 판단하기 쉽습니다.
실제로 원본 로그 Agent는 호출 실패 대상으로 찍힌 `ledger-service`를 1순위로 두었습니다.

반면 원본 로그 + 운영 맥락 Agent는 `payment-service`의 config diff를 추가로 읽었습니다.
Annotation Agent는 incident, payment deployment, telemetry object 사이의 관계를 먼저 따라갔습니다.
두 방식 모두 “어느 서비스 호출이 실패했는가”만 보지 않고 “그 시점에 어떤 서비스 배포와 설정 변경이 있었는가”를 함께 보게 되었고, 원인 배포 service인 `payment-service`를 1순위로 판단했습니다.

이 차이는 S3 Annotations가 raw log보다 “더 똑똑한 데이터”라는 뜻은 아닙니다.
오히려 raw log와 배포 맥락이 함께 있을 때, **어떤 객체와 어떤 context를 어떤 형태로 연결해서 볼 것인가**가 RCA 결과와 token 사용량에 영향을 줄 수 있다는 뜻에 가깝습니다.

<br>

## <a href="#design-note">🧯 Signal은 정답 힌트가 아니다</a><a id="design-note"></a>

`telemetry.signal`을 설계할 때는 조심해야 합니다.
`DownstreamTimeout`, `fraud_timeout_count`처럼 incident-specific signal을 넣으면 Agent에게 정답 범위를 너무 직접적으로 알려주는 것처럼 보일 수 있습니다.

그래서 Annotation Context Layer에서도 generic signal family를 기본으로 사용했습니다.

```text
http_status
latency
downstream_call
error_count
```

이렇게 하면 `telemetry.signal`은 정답을 알려주는 필드가 아니라, Agent가 telemetry object를 해석할 때 참고하는 관찰 범주가 됩니다.
RCA에 결정적으로 작동해야 하는 것은 signal 이름 자체가 아니라 `context.relations`, factual config diff, telemetry summary의 조합입니다.

<br>

## <a href="#limits">⚠️ 해석과 한계</a><a id="limits"></a>

이 결과를 해석할 때는 몇 가지 한계를 함께 봐야 합니다.

첫째, 이번 실험에서는 장애가 발생한 시간 구간을 Agent에게 알려줬습니다.
즉 Agent가 "언제부터 언제까지의 로그를 볼지"까지 직접 찾아낸 것은 아닙니다.
실제 운영에서는 장애 시간 구간을 추정하는 단계도 필요합니다.

둘째, 장애 유형이 아직 좁습니다.
이번에는 payment service의 HTTP 요청 흐름에서 발생한 장애만 다뤘습니다.
queue backlog, DLQ, auth failure, data quality issue처럼 모양이 다른 장애에서도 같은 효과가 나는지는 더 봐야 합니다.

셋째, 이번 실험에서는 Annotation Context Layer가 읽은 log row 수가 Raw 계열보다 적지 않았습니다.
따라서 이 결과를 “로그 검색량을 줄였다”로 해석하면 안 됩니다.
이번 결과에서 확인한 것은 같은 telemetry 범위를 읽더라도, 관계 그래프와 signal summary를 쓰면 Agent에게 전달되는 context의 형태와 token 사용량이 달라진다는 점입니다.

<br>

## <a href="#small-models">🔍 더 작은 모델에서는 어떨까?</a><a id="small-models"></a>

같은 실험을 다른 모델에서도 돌려봤습니다.
여기서도 중요한 질문은 같습니다.
raw log만 볼 때, 운영 맥락을 그대로 붙일 때, Annotation Context Layer를 쓸 때 결과가 어떻게 달라지는가입니다.

| Model | 원본 로그 | 원본 로그 + 운영 맥락 | Annotation Context Layer |
| --- | ---: | ---: | ---: |
| Haiku 4.5 | 3/5 | 5/5 | 5/5 |
| Nova Micro | 3/5 | 2/5 | 4/5 |
| Nova 2 Lite | 3/5 | 5/5 | 5/5 |

Nova Micro에서는 운영 맥락을 더 줬는데도 B가 2/5에 그쳤습니다.
`payment-timeout`, `fraud-latency`, `fraud-error-rate`처럼 증상이 보이는 service와 실제 원인 service를 분리해야 하는 케이스에서 흔들렸습니다.
즉 작은 모델에서는 context를 많이 주는 것만으로는 충분하지 않았습니다.
주어진 config diff나 metric을 RCA ranking에 안정적으로 반영하지 못하면, raw log에 보이는 증상 쪽으로 다시 끌려갑니다.

반면 Nova 2 Lite에서는 원본 로그 + 운영 맥락과 Annotation Context Layer가 모두 5/5였습니다.
다만 token 사용량은 크게 달랐습니다.

<div class="language-text highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span style="display:inline-block; min-width: 340px;">Nova 2 Lite + 원본 로그 + 운영 맥락:</span><span style="display:inline-block; width: 140px; text-align: right;">223,466 tokens</span>
<span style="display:inline-block; min-width: 340px;">Nova 2 Lite + Annotation Context Layer:</span><span style="display:inline-block; width: 140px; text-align: right;">58,059 tokens</span></code></pre></div></div>

이 결과는 두 가지를 보여줍니다.

첫째, **Nova Micro처럼 작은 모델에서도 맥락을 구조화하는 것은 의미가 있었습니다.**
원본 로그 + 운영 맥락은 2/5에 그쳤지만, Annotation Context Layer는 4/5였습니다.
**같은 운영 맥락이라도 raw 형태로 길게 붙이는 것보다, incident와 연결된 service, deployment, telemetry 관계로 정리해 주는 편이 모델이 따라가기 쉬웠다고 볼 수 있습니다.**

둘째, 같은 정확도가 나온 조건에서는 context를 어떤 형태로 전달하느냐가 token 비용을 크게 갈랐습니다.
Nova 2 Lite에서는 원본 로그 + 운영 맥락과 Annotation Context Layer가 모두 5/5였지만, Annotation Context Layer가 훨씬 적은 token을 사용했습니다.
**즉 운영에서 작은 모델을 쓰려면 “더 많은 raw context를 넣는 방식”보다 “관계 그래프로 필요한 맥락을 구조화해서 넣는 방식”이 더 현실적인 선택지가 될 수 있습니다.**

<br>

## <a href="#outro">마무리</a><a id="outro"></a>

3편에서 확인하고 싶었던 것은 하나였습니다.

> Annotations 기반 Context Layer가 AI SRE Agent의 조사 흐름에 실제로 도움이 되는가?

이번 결과는 다음처럼 정리할 수 있습니다.
S3 Annotations는 raw log를 대체하지 않습니다.
그리고 Annotation 관계 그래프만 붙인다고 자동으로 token이 줄지도 않습니다.

하지만 S3 Annotations로 service, deployment, telemetry, runbook 사이의 관계를 잡으면 이야기가 달라집니다.
이번 실험에서는 Annotation Context Layer가 원본 로그 + 운영 맥락보다 약 25.0% 적은 token으로 같은 5/5 정확도를 냈습니다.
또 같은 telemetry 범위를 signal summary로 전달한 D 조건에서는 token을 51,023까지 줄였습니다.
이는 원본 로그 + 운영 맥락보다 약 49.0% 적은 수치입니다.

즉 Annotations 기반 Context Layer의 가치는 "정답을 annotation에 넣는 것"이 아니라, **Agent가 읽어야 할 운영 맥락을 관계 그래프와 signal summary로 구조화해 제공하는 것**에 있습니다.

1편에서는 OKF 문서를 흉내냈고, 2편에서는 S3 객체 전반의 의미 계층을 정리했습니다.
3편에서는 그 의미 계층을 AI SRE 시나리오로 검증했습니다.
아직 더 큰 로그 corpus, window inferred 조건, queue/batch/security profile 검증이 남아 있습니다.
다음에는 회사에서 운영되는 실제 서비스에 적용해보고 후기를 약속드리며 글을 마칩니다.

<br>

## <a href="#references">References</a><a id="references"></a>

- [Amazon S3 Annotations overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/annotations-overview.html)
- [S3 Metadata annotation table schema](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-annotation-schema.html)
- [S3 Metadata tables overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-overview.html)
- [Use an AgentCore gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-using.html)
- [Amazon Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [Strands Agents documentation](https://strandsagents.com/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)

---

{% include s3_annotations.html %}
