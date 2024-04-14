---
layout: post
current: post
navigation: True
title: LLM Evaluation에 대해 끄적이기
date: 2024-03-31 00:00:00
tags: [ai, genai, llm]
class: post-template
subclass: "post tag-ai"
author: HeuristicWave
---

LLMOps, LLM App Development Life Cycle의 한 부분 LLM Evaluation에 대하여...

# Intro

LLM Evaluation 솔루션에 대해 조사를 하다, 도입부에 소개할 만한 좋은 글을 발견했습니다.

[📝 What the History of Software Development Tells Us about the Hurdles to Enterprise Adoption of LLMs](https://maverickventures.medium.com/what-the-history-of-software-development-tells-us-about-the-hurdles-to-enterprise-adoption-of-llms-c96bc968456d)

> _소프트웨어 개발 프로세스에 모든 단계가 중요하지만, 궁극적인 목표는 사용자에게 안정적으로 가치를 제공하는 것이므로 '테스트', '평가', '모니터링'은 항상 역사적으로 큰 시장이었습니다.
> 자체 설문 조사에서 LLM을 운영에 배포하는 데 있어 가장 큰 장벽은 "모델 품질 평가(evaluating model quality)"를 꼽았습니다._

**LLM Evaluate, Test, Monitoring Landscape**

_아래 그림은 위에서 언급한 글에 소개된 LLMOps Landscape로, 잘못된 내용들이 있으므로 참고만 하세요._

<img src="https://miro.medium.com/v2/resize:fit:4800/format:webp/1*7ot7nNqNWIu7Tw8X8yyGsw.png" width="870"/>

이번 포스팅에서는 해당 글에서 소개하는 여러 LLM 평가 스타트업 중, 'UpTrain'과 'Ragas'를 중점으로 이야기를 풀어나가 보겠습니다.

<br>

## 1️⃣ [UpTrain](https://uptrain.ai/)

### 주요 특징

- [Y Combinator](https://www.ycombinator.com/companies/uptrain-ai)
- [GitHub(1.9k)](https://github.com/uptrain-ai/uptrain)
- Ground Truth 비교, 사용자 정의 평가 등 9개의 범주 아래, 20개의 평가 메트릭을 제공
- LlamaIndex, Langfuse, ChromaDB 등에 대한 통합 제공
- 홈페이지에서 GDPR, ISO 27001 취득한 것으로 보임

### 사용 방법

#### LLM 모델 선택

`uptrain`을 설치하고 다음과 같이 모델을 선택합니다. 본래 모델 선택 시, `EvalLLM()`에 LLM 호출을 위한 KEY를 기재합니다.
Amazon Bedrock 서비스를 사용하는 경우, Settings를 활용해 LLM 주입이 가능합니다. ([참고](https://github.com/uptrain-ai/uptrain/issues/589))

```python
from uptrain import Settings, EvalLLM, Evals
import json

settings = Settings(model="bedrock/anthropic.claude-3-sonnet-20240229-v1:0")
eval_llm = EvalLLM(settings=settings)
```

#### 평가 DataSet 준비

아쉽게도 UpTrain에서는 평가 DataSet 생성을 보조하는 기능이 아직은 없습니다. 질문(`question`), 맥락(`context`), 답변(`response`)을 다음과 같이 준비합니다.

```python
data = [{
    "question": [
        "Which is the most popular global sport?",
        "Who created the Python language?",
    ],
    "context": [
        "_comment" : "생략"
    ],
    "response":[
        "Football is the most popular sport with around 4 billion followers worldwide",
        "Python language was created by Guido van Rossum.",
    ]
}]
```

#### 평가 메트릭 선택

제공되는 [지표](https://docs.uptrain.ai/predefined-evaluations/overview)를 참고하여, `checks` 변수에 list로 주입합니다.

```python
results = eval_llm.evaluate(
    data=data,
    checks=[Evals.CONTEXT_RELEVANCE, Evals.FACTUAL_ACCURACY, Evals.RESPONSE_COMPLETENESS]
)

# Results
print(json.dumps(results, indent=3))
```

<details><summary markdown="span">결과 보기 👉 Click</summary>

```json
[
  {
    "question": [
      "_comment" : "생략"
    ],
    "context": [
      "_comment" : "생략"
    ],
    "response": [
      "_comment" : "생략"
    ],
    "score_context_relevance": 1.0,
    "explanation_context_relevance": "{\n    \"Reasoning\": \"
    The extracted context contains two separate paragraphs, each addressing one of the two queries. \
    The first paragraph discusses the popularity of sports globally, \
    mentioning that football is considered the most popular sport with a large following. \
    This information can answer the first query 'Which is the most popular global sport?' completely. \
    The second paragraph provides details about the creation of the Python programming language by Guido van Rossum, \
    which can answer the second query 'Who created the Python language?' completely. \
    Therefore, the extracted context can answer both queries in their entirety.\",\n    \"Choice\": \"A\"\n}",
    "score_factual_accuracy": 1.0,
    "explanation_factual_accuracy": "{\n    \"Result\": [\n        {\n
    \"Fact\": \"Football is the most popular sport with around 4 billion followers worldwide.\",\n
    \"Reasoning\": \"The context directly states that 'Football is undoubtedly the world's most popular sport' \
    and mentions that it has 'a followership of more than 4 billion people'. \
    This supports the given fact.\",\n            \"Judgement\": \"yes\"\n        },\n
    {\n            \"Fact\": \"Python language was created by Guido van Rossum.\",\n
    \"Reasoning\": \"The context explicitly mentions that 'Python, created by Guido van Rossum in the late 1980s,
    is a high-level general-purpose programming language'. This directly supports the given fact.\",\n
    \"Judgement\": \"yes\"\n        }\n    ]\n}",
    "score_response_completeness": 1.0,
    "explanation_response_completeness": "_comment" : "생략"
  }
]
```

</details>

#### 대시보드 사용

UpTrain OSS Dashboard도 제공하는데, Docker Compose로 Server Up 하는 스크립트 실행합니다.
이어서 `localhost:3000`에서 코드로 평가를 진행할 때와 동일하게 GUI를 눌러 수행하면 평가 결과를 시각화하여 볼 수 있습니다.

![dashboard](/assets/built/images/post/etc/uptrain.png)

### Pros and Cons

**장점**

- 다양한 상황에서 적용할 수 있는 지표들에 대한 손쉬운 적용
- Evaluations/Prompts 테스트에 대하여 한 번에 여러 가지 테스트가 가능 (배치)
- ‘A/B 테스트’, ‘사용자 정의 프롬프트 기반 평가 셋(사용자가 평가 기준을 LLM에게 전달)’ 등의 기능 제공
- Local 환경에서 평가 가능

**단점**

- GitHub에서 제공하는 OSS Dashboard가 오퍼링 웹사이트만큼 다양한 기능을 제공하고 있지는 않음
- 평가 후, 얻은 `jsonl` 파일을 업로드해서 시각화하는 기능은 미제공

### 🤔 생각

[UpTrain 팀의 수익화](https://news.ycombinator.com/item?id=35069839) 계획에서 더 넓은 통합과 Dashboard의 향상된 경험을 제공하는 것으로 보이나,
OSS로 활용한다면 별도의 Dashboard 개발 필요한 것 같습니다. 또한 아직은 통합을 지원하는 범위가 좁지만, 평가 역할 수행하기에는 좋은 도구인 것 같습니다.

LLM Evaluation은 어렵습니다. 어떻게 평가 기준을 설계해야 하는지 모르겠다면, 제공되는 20여 개의 metrics들로 다양한 시각에서 평가하는 방법을 고려할 수 있을 것 같습니다.
그뿐만 아니라 해당 도구를 채택하지 않더라도, 제공하는 metrics들을 참고하면 계획하고 있는 평가 방법들에 대한 좋은 참고 자료가 되는 것 같습니다.
예를 들어, UpTrain은 응답의 품질을 평가하기 위해 대응 여부, 간결성, 관련성, 유효성, 일관성 등 5가지의 요소로 품질을 평가합니다.
UpTrain이 제공하는 metrics의 종류는 평가 Task를 해결하기 위한 방법들이 되므로, 평가 계획 수립에 유용할 것 같습니다.

<br>

## 2️⃣ [Ragas](https://docs.ragas.io/en/stable/)

### 주요 특징

- [GitHub(4.1k)](https://github.com/explodinggradients/ragas)
- RAG 파이프라인 전용 평가 솔루션 : 데이터 셋 생성, RAG 평가, 모니터링 등의 기능 제공
- 정확도, 관련성 등 10개의 범주 아래 다양한 메트릭 제공(평가 방법에 대한 수식 제공)
- 언어별 다중 프롬프트 생성, 평가를 위한 테스트 데이터 증강 등 부가 기능 제공
- LangChain을 활용한 CSP 모델 및 LlamaIndex, LangSmith 등 다양한 프레임워크에 통합 지원
- Atina, Zeno, Tonic 등 다양한 방법으로 시각화 제공

### 🤔 생각

[UpTrain 문서에서 RAG 파이프라인을 분석하는 글](https://docs.uptrain.ai/tutorials/analyzing-failure-cases)을 보고, RAG 파이프라인 전용 평가 도구의 필요성을 고민하다 Ragas가 흥미로워 살펴보게 되었습니다.
UpTrain뿐만 아니라, 다른 Evaluation 솔루션들도 'RAG 평가'는 방법 중 하나일 뿐, 왜 RAG에 한정하여 전문적인 도구가 필요할지 고민해 보았습니다.
Ragas는 **[MDD(지표 중심 개발)](https://docs.ragas.io/en/stable/concepts/metrics_driven.html)**라는 용어를 내세우며, LLM 앱을 데이터 기반 의사 결정이 매우 중요하다고 강조합니다.
해당 사실은 Ragas뿐만 아니라, 다른 Evaluation 솔루션들도 입을 모아 Observability의 중요성을 언급합니다.
그러나 다른 솔루션들의 문서와는 달리, Ragas 문서는 [측정 메트릭](https://docs.ragas.io/en/stable/concepts/metrics/index.html) 별, 수식과 예시와 함께 제공하니 그들의 주장에 조금 더 마음이 가는 것 같습니다.

Ragas의 테스트 데이터 셋 생성 기능은 Ragas를 채택하지 않더라도 도움이 될 것 같습니다. 더하여 Ragas는 내부적으로 langchain을 활용하므로,
프로덕션급 LLM 앱 구축 플랫폼인 [LangSmith](https://docs.smith.langchain.com/)를 보완하여 더 나은 기능을 제공할 것으로 보입니다. (앞으로의 숙제네요 🫠)

<br>

## Outro

Evaluation에서도 Data가 매우 중요합니다. 아직 스터디가 더 필요하지만, 그동안 '평가 Data 제작' 부분은 종종 봤지만, '평가 Data 활용 방안'은 더 적은 것 같습니다.
평가 **Data의 재사용성**을 높이기 위해 고려할 지점이 있어 보여, 다음과 같이 몇 자 끄적이며 마치도록 하겠습니다.

- **Training** : sLM 도입 및 파인튜닝 시, 튜닝을 위한 Datasets에 평가 Datasets을 활용할 수 있음
- **호환성** : 다양한 평가도구들이 `jsonl` 형태로 지원하여, 이기종 간 호환이 자유롭다면 다양한 평가 가능

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해주세요! 😃

---
