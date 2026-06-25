---
layout: post
current: post
cover: assets/built/images/background/s3-annotations-semantic-layers.svg
navigation: True
title: "Amazon S3 Annotations 2편: 객체에 의미 계층 붙이기"
display_title: "Amazon S3 Annotations 2편:<br>객체에 의미 계층 붙이기"
date: 2026-06-25 00:00:00
tags: [aws, genai, lab]
class: post-template
subclass: "post tag-ai"
author: HeuristicWave
---

이번 글에서는 S3 Annotations를 특정 예제의 부가 정보 저장소가 아니라, S3 객체 전반에 의미 계층을 붙이는 context layer로 보는 관점을 정리합니다.

[1편](/S3-OKF-Annotations.html)에서는 S3 Annotations로 Open Knowledge Format을 흉내내는 실험을 했습니다.
Markdown 문서를 S3 Object로 올리고, YAML frontmatter, Markdown links, summary를 각각 `okf.frontmatter`, `okf.links`, `okf.summary` annotation으로 분리했습니다.

다만 S3 bucket에 들어가는 것은 Markdown 문서만이 아닙니다.
이미지, 오디오, PDF, 로그, Parquet, 모델 산출물, 임베딩 참조, 거버넌스 정책까지 모두 객체로 존재할 수 있습니다.
이 객체들을 다루기 시작하면 S3 Annotations는 "특정 문서 포맷의 metadata 저장소"라기보다 **S3 객체에 의미 계층을 붙이는 context layer**에 가까워집니다.

여기서 자연스럽게 S3 Metadata tables와의 관계도 다시 봐야 합니다.
S3 Metadata tables가 객체의 상태와 변경 이력을 자동으로 테이블화한다면, S3 Annotations는 사람이 또는 시스템이 직접 붙이는 의미를 객체 옆에 둡니다.
그리고 S3 Metadata annotation table은 그 의미를 Athena 같은 분석 엔진에서 조회할 수 있게 만드는 인덱스 역할을 합니다.

이 관점에서 보면 이번 글의 핵심은 두 가지입니다.

- S3 Metadata tables는 객체 상태와 변경 이력을 자동으로 테이블화하고, S3 Annotations는 객체에 사람이 또는 시스템이 직접 의미를 붙입니다.
- Markdown 문서를 넘어 이미지, 오디오, PDF, 데이터셋까지 다루려면 `asset.*`, `analysis.*`, `agent.*`, `governance.*` 같은 역할별 namespace가 필요합니다.

<br>

## <a href="#metadata-vs-annotations">🤔 S3 Metadata와 S3 Annotations는 무엇이 다른가</a><a id="metadata-vs-annotations"></a>

S3 Annotations를 정확히 이해하려면 먼저 S3에서 말하는 metadata, Metadata tables, annotation table의 범위를 잡고 가야 합니다.
세 용어가 모두 "객체를 설명하는 정보"처럼 보이지만, 실제로는 붙는 위치와 조회 방식이 다릅니다.

- **[S3 object metadata](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html)**는 객체 자체에 붙어 있는 system-defined metadata와 user-defined metadata입니다.
- **[S3 Metadata tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-overview.html)**는 그 객체 metadata와 변경 이벤트를 read-only Apache Iceberg table로 자동 수집해 조회하게 해주는 기능입니다.
- **[S3 Annotations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/annotations-overview.html)**는 객체를 다시 업로드하지 않고도 객체 버전 옆에 named payload를 따로 붙이는 기능입니다.

![S3 Metadata tables와 S3 Annotations 관계](/assets/built/images/post/ai/s3-annotations/metadata-vs-annotations.svg)

위 그림에서 중요한 점은 S3 Annotations가 S3 object metadata 안에 들어가는 필드가 아니라는 점입니다.
Annotations는 객체 버전 옆에 따로 붙고, S3 Metadata tables의 annotation table을 통해 조회할 수 있습니다.

공식 문서 기준으로 S3 Metadata tables는 기본적으로 system-defined object metadata, object tag, upload 시점의 user-defined metadata를 수집하고, 객체가 언제 수정되거나 삭제되었는지 같은 event metadata도 제공합니다.
여기서 말하는 `user_metadata`는 실제 schema에 있는 컬럼이고, `requester`, `source_ip_address`, `request_id`, `record_type`, `record_timestamp` 같은 값이 event 쪽을 설명합니다.

표로 비교하면 다음과 같습니다.

<table style="display:table; width:100%; table-layout:fixed; overflow-x:visible; white-space:normal; overflow-wrap:anywhere;">
  <colgroup>
    <col style="width:18%;">
    <col style="width:41%;">
    <col style="width:41%;">
  </colgroup>
  <thead>
    <tr>
      <th>구분</th>
      <th>S3 Metadata tables</th>
      <th>S3 Annotations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>본질</td>
      <td>S3 객체 상태와 변경 이벤트를 queryable table로 제공</td>
      <td>객체 버전에 named payload를 붙임</td>
    </tr>
    <tr>
      <td>생성 주체</td>
      <td>Amazon S3가 객체와 이벤트에서 자동 수집</td>
      <td>애플리케이션, Agent, ML pipeline, 운영 시스템</td>
    </tr>
    <tr>
      <td>수정 방식</td>
      <td>Metadata table 자체는 read-only</td>
      <td>전용 annotation API로 생성/수정/삭제</td>
    </tr>
    <tr>
      <td>대표 데이터</td>
      <td>object key, size, storage class<br>tag, user metadata<br>event type, requester</td>
      <td>label, transcript, lineage<br>processing result<br>compliance record</td>
    </tr>
    <tr>
      <td>조회 테이블</td>
      <td>journal table<br>live inventory table<br>annotation table</td>
      <td>S3 Metadata annotation table</td>
    </tr>
    <tr>
      <td>답하는 질문</td>
      <td>"버킷에 무엇이 있고 어떻게 변했나?"</td>
      <td>"이 객체를 어떤 맥락으로 읽어야 하나?"</td>
    </tr>
  </tbody>
</table>

S3 Annotations는 S3 object metadata의 하위 필드가 아니라, 객체 옆에 붙는 별도 context payload입니다.
User-defined metadata는 업로드 시점의 header에 가깝고, 바꾸려면 객체 copy가 필요하지만, annotation은 별도 API로 붙이고 S3 Metadata annotation table을 통해 조회할 수 있습니다.

예를 들어 `assets/audio/music.m4a`라는 객체가 있다고 해보겠습니다.
S3 Metadata tables는 이 객체의 상태를 잘 설명합니다.

```text
key = assets/audio/music.m4a
size = 8MB
storage_class = STANDARD
encryption_status = SSE-S3
last_modified_date = ...
```

하지만 객체를 실제로 활용하려면 상태만으로는 부족합니다.
다음은 객체의 의미와 사용 맥락에 가까운 질문입니다.

```text
이 오디오 파일은 어떤 캠페인과 연결되는가?
이 파일의 transcript는 어디에 있는가?
이 asset은 어떤 OKF 문서가 설명하는가?
Agent가 이 파일을 사용할 때 어떤 retrieval hint를 따라야 하는가?
이 파일은 내부 전용인가, 외부 공유 가능한가?
```

이런 맥락은 S3 Annotations로 객체 옆에 붙이는 편이 자연스럽습니다.

```text
asset.media
analysis.labels
analysis.transcript
analysis.embedding_ref
governance.policy
context.relations
agent.retrieval_hint
```

지금까지의 관계를 짧게 정리하면 다음과 같습니다.

```text
S3 Metadata tables
  ├─ journal table        # 객체 변경 이벤트
  ├─ live inventory table # 현재 객체 상태
  └─ annotation table     # S3 Annotations payload를 queryable하게 만든 테이블
```

즉 S3 Annotations가 S3 Metadata tables를 대체하는 것은 아닙니다.
S3 Annotations는 객체 옆에 context를 붙이는 기능이고, S3 Metadata annotation table은 그 context를 대량 조회하게 해주는 테이블입니다.
여기까지 정리하고 나면 다음 문제는 자연스럽게 "annotation 이름을 어떻게 나눌 것인가"가 됩니다.

<br>

## <a href="#namespace">🧩 Context Layer 만들기</a><a id="namespace"></a>

[AWS의 S3 Annotations 소개 글](https://aws.amazon.com/ko/blogs/aws/amazon-s3-annotations-attach-rich-queryable-context-directly-to-your-objects/)은 S3 객체 옆에 rich, queryable context를 붙일 수 있다는 방향을 보여줍니다.
다만 그 글이 의미 계층을 어떻게 나눌지에 대한 Best Practice나 청사진까지 제시하는 것은 아닙니다.
그래서 여기서는 S3 Annotations로 의미 계층을 만든다면 namespace를 어떻게 나눌 수 있을지 작게 시도해보겠습니다.

S3 버킷에는 Markdown 문서만 있는 것이 아닙니다.
이미지, 오디오, PDF, Parquet, 모델 산출물처럼 서로 다른 성격의 객체가 함께 존재할 수 있습니다.
이 객체들에 붙는 annotation까지 하나의 context layer로 다루려면, annotation 이름은 파일 형식보다 payload의 책임을 더 잘 드러내야 합니다.

그래서 이 글에서는 namespace를 "어떤 포맷에서 왔는가"보다 "무엇을 설명하는가" 기준으로 나눠보겠습니다.

```text
okf.*          # OKF 문서 구조
asset.*        # 파일/미디어 자체 정보
analysis.*     # 이미지/오디오/ML 분석 결과
governance.*   # 민감도, 보존, 접근 정책
agent.*        # Agent 검색/라우팅 단서
context.*      # 객체 간 관계
```

각 namespace의 책임은 다음처럼 생각할 수 있습니다.

| Namespace | 역할 | 예시 |
| --- | --- | --- |
| `okf.*` | OKF 문서 포맷 | `okf.frontmatter`, `okf.summary`, `okf.links` |
| `asset.*` | 객체 자체의 설명 | `asset.media`, `asset.caption` |
| `analysis.*` | 분석 결과 | `analysis.labels`, `analysis.transcript`, `analysis.embedding_ref` |
| `governance.*` | 정책과 통제 | `governance.policy`, `governance.retention` |
| `agent.*` | Agent 검색/라우팅 단서 | `agent.summary`, `agent.retrieval_hint`, `agent.routing` |
| `context.*` | 객체 간 관계 | `context.relations` |

이렇게 나누면 S3 bucket 안의 다양한 객체를 하나의 의미 그래프로 묶을 수 있습니다.

여기서 중요한 것은 namespace를 "누가 읽을 것인가"가 아니라 "무엇을 책임지는가" 기준으로 나누는 것입니다.
예를 들어 `agent.summary`가 편하다고 해서 민감도나 보존 정책까지 `agent.*`에 넣기 시작하면, Agent 검색 단서와 governance policy가 섞입니다.
반대로 `governance.*`는 정책과 통제만 담당하고, `agent.*`는 Agent가 참고할 단서만 담당하게 하면 같은 객체를 여러 도구가 읽어도 역할이 덜 꼬입니다.
이제 이 namespace 중에서 관계를 다루는 `context.relations`를 조금 더 자세히 보겠습니다.

<br>

## <a href="#relations">🔗 context.relations로 관계 만들기</a><a id="relations"></a>

Context layer를 만들 때 중요한 것은 개별 객체의 설명만 붙이는 것이 아닙니다.
객체와 객체 사이의 관계도 함께 표현해야 합니다.
`context.relations`는 이 역할을 맡는 중립적인 relation layer로 둘 수 있습니다.

이 관계는 Markdown 문서만 대상으로 하지 않습니다.
이미지, 오디오, PDF, 데이터셋, 모델 산출물처럼 S3에 있는 어떤 객체도 연결 대상이 될 수 있습니다.

예를 들어 다음 두 객체가 있다고 가정합니다.

```text
okf/music/campaign-theme.md
assets/audio/music.m4a
```

이 둘은 본문 포맷은 다르지만, `context.relations`로 link 관계를 맺을 수 있습니다.

```json
{
  "relations": [
    {
      "type": "describes",
      "target": "assets/audio/music.m4a",
      "target_kind": "asset",
      "label": "campaign theme audio"
    }
  ]
}
```

필요하면 반대쪽 객체에는 `described_by` 같은 역방향 relation을 둘 수 있습니다.
핵심은 두 객체가 같은 파일 형식일 필요가 없다는 점입니다.

```text
okf/music/campaign-theme.md
  -- context.relations: describes -->
assets/audio/music.m4a

assets/audio/music.m4a
  -- context.relations: described_by -->
okf/music/campaign-theme.md
```

즉 `context.relations`는 특정 포맷의 링크 추출 결과가 아니라, S3 객체 전체를 연결하는 context graph의 가장 작은 단위입니다.

<br>

## <a href="#use-cases">🧭 어디에 쓸 수 있을까</a><a id="use-cases"></a>

`context.relations`는 여러 포맷의 객체를 하나의 주제로 묶을 때 가장 이해하기 쉽습니다.
예를 들어 월드컵 캠페인 관련 자료가 S3에 흩어져 있다고 해보겠습니다.

<img src="/assets/built/images/post/ai/s3-annotations/worldcup-context-relations.svg" alt="World Cup context relations" style="display:block; width:88%; max-width:920px; margin:1.5em auto;">

각 객체의 본문은 Markdown, 영상, 이미지, 오디오, Parquet처럼 서로 다릅니다.
하지만 사용자나 Agent 입장에서는 모두 "월드컵 캠페인"이라는 같은 맥락의 자료입니다.
이때 캠페인 overview 문서에 `context.relations`를 붙여 관련 객체를 묶을 수 있습니다.

이렇게 해두면 원본 파일을 한 곳으로 복사하지 않아도 됩니다.
각 객체는 자기 위치에 그대로 두고, 관계만 annotation으로 붙입니다.
검색 화면이나 Agent는 먼저 annotation table에서 월드컵 overview 문서를 찾고, 그 문서의 `context.relations`를 펼쳐 관련 영상, 이미지, 음원, 데이터셋을 함께 보여줄 수 있습니다.

이 패턴의 핵심은 "객체를 옮기지 않고 의미 계층만 얹는다"는 점입니다.
Media Asset 관리에서는 캠페인 문서, 원본 영상, 썸네일 이미지, 배경 음원, 자막 파일을 같은 asset group으로 묶을 수 있습니다.
검색 화면은 객체 본문을 모두 열지 않아도 `context.relations`를 따라가며 관련 자료를 함께 보여줄 수 있습니다.

AI SRE 쪽으로 옮기면 같은 패턴을 로그와 운영 문서에도 적용할 수 있습니다.
예를 들어 `incidents/payment-5xx.md`가 특정 시간대의 log parquet, metric export, deployment event, runbook을 `context.relations`로 참조하게 만들 수 있습니다.
그러면 Agent는 raw log를 처음부터 훑기보다, 먼저 incident summary가 연결한 telemetry와 runbook을 따라가며 조사 범위를 좁힐 수 있습니다.

중요한 점은 이미지, 오디오, 로그 원문을 annotation payload에 넣는 것이 아닙니다.
S3 object는 그대로 두고, annotation에는 "이 객체가 무엇과 연결되는가"라는 얇은 의미 계층만 둡니다.
객체의 storage class나 size는 S3 Metadata tables가 알려주고, 이 객체가 어떤 캠페인, 문서, 로그, 정책과 연결되는지는 `context.relations`가 설명합니다.
여기까지가 의미 계층을 만드는 방식이라면, 운영에서는 곧바로 보안 문제가 따라옵니다.

<br>

## <a href="#permissions">🔐 객체 권한과 annotation 권한을 나눠서 보기</a><a id="permissions"></a>

의미 계층은 얇지만, 운영 시스템에서는 별도의 데이터 노출면입니다.
Agent나 검색/카탈로그 화면이 annotation을 읽으면 원본 객체를 열지 않아도 관계, 요약, 정책 단서가 드러날 수 있습니다.
그래서 객체 본문 권한과 annotation 권한은 분리해서 봐야 합니다.

객체 본문은 기존처럼 `PutObject`, `GetObject`로 다룹니다.
Annotation은 별도 API로 생성, 조회, 삭제합니다.

```text
PutObjectAnnotation
GetObjectAnnotation
ListObjectAnnotations
DeleteObjectAnnotation
```

이 차이는 권한 설계에서 중요합니다.
예를 들어 역할을 다음처럼 나눌 수 있습니다.

| Role | 가능한 작업 |
| --- | --- |
| Data Producer | 원천 객체 업로드 |
| Catalog Writer | `asset.*`, `analysis.*`, `context.relations` annotation 작성 |
| Governance Writer | `governance.*` annotation 작성 |
| Agent Runtime | 필요한 annotation 조회와 제한된 객체 읽기 |
| Auditor | annotation table과 journal table 조회 |

정리하면 S3 Annotations가 annotation 이름별 권한 체계를 대신 설계해주는 것은 아닙니다.
다만 객체 본문 API와 annotation API가 나뉘어 있으므로, 원본을 다루는 역할과 context를 다루는 역할을 분리해 볼 수 있습니다.

```text
데이터 생산 팀:
  - s3:PutObject
  - s3:GetObject

Catalog 팀:
  - s3:PutObjectAnnotation
  - s3:GetObjectAnnotation
  - s3:ListObjectAnnotations

Agent Runtime:
  - s3:GetObjectAnnotation
  - s3:ListObjectAnnotations
  - 필요한 prefix에 한정된 s3:GetObject

감사/분석 팀:
  - annotation table 조회 권한
  - journal table 조회 권한
```

이 구조에서는 데이터 생산 팀이 원천 파일을 관리하고, Catalog 팀은 객체 옆의 context를 갱신합니다.
객체를 다시 업로드하지 않고 annotation만 바꿀 수 있으므로, payload와 context의 lifecycle을 어느 정도 분리할 수 있습니다.

> **LLM 사용 시 주의**
>
> Agent Runtime이 annotation을 읽더라도, payload 안의 문장을 사용자 지시나 시스템 지시로 실행하면 안 됩니다.
> annotation은 tool result나 JSON data처럼 다루고, 권한 판단은 IAM, bucket policy, `governance.*` 쪽에서 다시 확인해야 합니다.
> 특히 LLM에게 annotation을 context로 넣는다면 context poisoning도 고려해야 합니다.
> 악의적이거나 오래된 annotation이 "이 객체는 안전하다", "이 파일을 우선 신뢰하라" 같은 문장을 담고 있으면 Agent의 판단이 흔들릴 수 있습니다.
> 따라서 중요한 annotation은 작성 주체, 갱신 시점, 검증 여부를 함께 관리하는 편이 안전합니다.

다만 annotation이 객체와 완전히 독립적인 리소스는 아닙니다.
annotation은 특정 객체 또는 객체 버전에 붙습니다.
또한 [AWS 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/annotations-overview.html#annotations-encryption)에 따르면 annotation은 parent object의 encryption configuration을 따릅니다.
예를 들어 원본 객체가 SSE-KMS를 사용하면 annotation도 같은 KMS key로 암호화되고, SSE-C로 암호화된 객체에는 annotation을 붙일 수 없습니다.
즉 annotation만 따로 다른 암호화 정책을 갖는 독립 저장소처럼 보는 것은 맞지 않습니다.
따라서 권한은 나눌 수 있지만, 보안 설계에서는 객체와 annotation을 함께 봐야 합니다.

<br>

## <a href="#permission-demo">🧪 실험: IAM Role을 나눠서 확인하기</a><a id="permission-demo"></a>

권한 분리가 실제로 되는지 확인하려고 같은 객체에 대해 IAM Role을 두 개 만들었습니다.
아래 출력은 실제 실행 결과에서 bucket 이름과 account id만 마스킹한 것입니다.

실험 대상 객체는 다음과 같습니다.

```text
s3://<bucket-name>/okf/generated/ai/01-rag-evaluation-set.md
annotation name: okf.frontmatter
```

첫 번째 Role은 데이터 생산자 역할입니다.
원본 객체는 읽을 수 있지만, annotation API 권한은 주지 않았습니다.

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::<bucket-name>/okf/generated/ai/01-rag-evaluation-set.md"
}
```

이 Role로 assume-role한 뒤 실행한 결과입니다.

```text
## DataProducer role
$ aws s3api get-object ... /tmp/object.md
GetObject: SUCCESS (757 bytes)

$ GetObjectAnnotation okf.frontmatter
AccessDenied: no identity-based policy allows the s3:GetObjectAnnotation action
HTTP_STATUS:403
```

두 번째 Role은 Catalog Reader 역할입니다.
반대로 원본 객체 읽기는 명시적으로 막고, annotation 조회만 허용했습니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<bucket-name>/okf/generated/ai/01-rag-evaluation-set.md"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectAnnotation",
        "s3:ListObjectAnnotations"
      ],
      "Resource": "arn:aws:s3:::<bucket-name>/okf/generated/ai/01-rag-evaluation-set.md"
    }
  ]
}
```

이 Role로 assume-role한 결과는 반대로 나왔습니다.

```text
## CatalogReader role
$ aws s3api get-object ... /tmp/object.md
AccessDenied: explicit deny in an identity-based policy

$ GetObjectAnnotation okf.frontmatter
{
  "type": "Context",
  "title": "RAG Evaluation Set",
  "description": "A small benchmark for retrieval quality and answer grounding.",
  "domain": "ai",
  "language": "en",
  "owner": "ai-context-team",
  "resource": "context://ai/rag-evaluation-set",
  "tags": ["ai", "en", "context"],
  "status": "experimental",
  "sensitivity": "internal",
  "timestamp": "2026-06-24T00:00:00Z"
}
HTTP_STATUS:200
```

이 결과가 보여주는 것은 명확합니다.
원본 객체를 읽을 수 있는 권한과 annotation을 읽을 수 있는 권한은 같은 것이 아닙니다.
Data Producer는 객체 본문을 읽었지만 annotation은 읽지 못했고, Catalog Reader는 객체 본문을 읽지 못했지만 annotation payload는 읽었습니다.

따라서 annotation은 단순한 보조 필드가 아니라 별도의 데이터 노출면으로 봐야 합니다.
Agent나 검색/카탈로그 화면에 annotation 조회 권한을 줄 때는, 원본 객체 권한과 별개로 어떤 annotation payload가 노출되는지 검토해야 합니다.
민감한 원문은 객체에 두고, annotation에는 검색과 판단에 필요한 최소한의 context만 넣는 편이 안전합니다.

<br>

## <a href="#limits">⚠️ 주의할 점</a><a id="limits"></a>

실험까지 해보니 S3 Annotations를 볼 때 경계가 조금 더 분명해졌습니다.
S3 Annotations는 객체 옆에 queryable context를 붙이는 기능이지, 검색엔진이나 Graph DB나 권한 모델을 자동 설계해주는 기능은 아닙니다.

첫째, annotation table은 검색엔진이 아닙니다.
Athena나 Iceberg-compatible engine으로 대량 조회하기에는 좋지만, 즉시 검색 UX, 랭킹, 오타 보정, 의미 기반 검색이 필요하면 OpenSearch나 vector DB를 별도로 봐야 합니다.

둘째, `context.relations`는 graph의 시작점이지 Graph DB 자체는 아닙니다.
1-hop, 2-hop 관계를 펼쳐 보는 정도는 Athena에서도 가능하지만, 여러 단계를 따라가며 관계를 탐색하거나 그래프 전체의 중요한 객체를 찾는 용도라면 Neptune 같은 graph DB가 더 맞습니다.

셋째, annotation table은 비동기입니다.
S3 Metadata table은 backfilling과 refresh 시간이 있으므로, 방금 쓴 annotation이 Athena 조회 결과에 바로 보이지 않을 수 있습니다.
운영 UI에서 "방금 붙인 annotation을 즉시 확인"해야 한다면 `GetObjectAnnotation` 같은 direct API와 annotation table 조회를 나눠 써야 합니다.

넷째, annotation은 LLM에게는 신뢰 경계입니다.
Agent가 annotation을 근거로 판단한다면 annotation을 바꾸는 행위는 객체 해석을 바꾸는 행위가 됩니다.
따라서 LLM에는 annotation을 지시문이 아니라 data/tool result로 전달하고, 권한 판단은 IAM, bucket policy, `governance.*` 같은 검증된 경로에서 다시 확인해야 합니다.

<br>

## <a href="#outro">Outro</a><a id="outro"></a>

1편에서는 S3 Annotations를 작은 문서 예제로 실험했습니다.
2편에서는 그 범위를 넓혀, S3 객체 전반에 context layer를 붙이는 관점으로 다시 보았습니다.

핵심은 다음과 같습니다.

```text
S3 Metadata tables = 객체 상태와 변경 이력을 queryable table로 제공
S3 Annotations = 객체에 사람이/시스템이 직접 의미 context를 붙이는 기능
Annotation table = 그 context를 Athena에서 대량 조회하는 인덱스
```

그리고 설계의 중심은 namespace입니다.

```text
asset.*        # 객체 자체 설명
analysis.*     # 이미지/오디오/ML 분석
governance.*   # 정책과 보존
agent.*        # Agent 검색/라우팅 단서
context.*      # 객체 간 관계
```

정리하면 S3 Annotations는 단순히 "S3에 metadata를 조금 더 붙이는 기능"이라기보다, 원본 객체는 그대로 두고 객체 옆에 의미 계층을 붙이는 방식에 가깝습니다.
Media Asset 관리에서는 캠페인, 영상, 이미지, 음원, 자막을 같은 맥락으로 묶을 수 있고, AI SRE에서는 incident 문서와 로그, metric, runbook을 연결하는 context layer로 쓸 수 있습니다.

다만 운영에서는 몇 가지 선을 분명히 그어야 합니다.
namespace는 payload의 책임 기준으로 나누고, 원본 객체 권한과 annotation 권한은 분리해서 봐야 합니다.
대량 탐색은 annotation table이 편하지만 비동기이고, 단일 객체의 최신 context는 `GetObjectAnnotation`이 더 직접적입니다.
LLM이나 Agent에게 annotation을 넘길 때는 지시문이 아니라 data로 취급하고, 보안 판단은 IAM, bucket policy, `governance.*` 같은 검증된 경로에서 다시 확인해야 합니다.

이 선을 지킬 수 있다면 S3 Annotations는 S3 객체를 Agent와 Analytics가 다루기 좋은 의미 단위로 바꾸는 가벼운 context layer가 될 수 있습니다.
다만 이것이 실제 운영 문제에서도 유의미한지는 별도로 확인해야 합니다.
3편에서는 AI SRE 예시로 incident 문서, 로그, metric, runbook을 연결해보고, S3 Annotations 기반 context layer가 Agent의 조사 흐름에 실제로 도움이 되는지 살펴보겠습니다.

<br>

## <a href="#references">References</a><a id="references"></a>

- [Amazon S3 Annotations overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/annotations-overview.html)
- [Working with object metadata](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html)
- [S3 Metadata tables overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-overview.html)
- [S3 Metadata live inventory table schema](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-inventory-schema.html)
- [S3 Metadata journal table schema](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-schema.html)
- [S3 Metadata annotation table schema](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-annotation-schema.html)
- [Example metadata table queries](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-example-queries.html)
- [Amazon S3 annotations launch blog](https://aws.amazon.com/blogs/aws/amazon-s3-annotations-attach-rich-queryable-context-directly-to-your-objects/)

---

{% include s3_annotations.html %}
