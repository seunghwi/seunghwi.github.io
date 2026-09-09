---
author_profile: true
categories:
- AI
layout: single
permalink: /topics/ai/llm-overview/
sidebar:
  nav: docs
tags:
- AI
- LLM
- Transformer
- RAG
- Fine-tuning
- MCP
- Agent
title: "LLM 전체 구조 : 학습부터 답변 생성, RAG와 Agent까지 한 번에이해하기"
toc: true
---



ChatGPT, Claude, Gemini, Llama 같은 **LLM(Large Language Model)**을
공부하다 보면 Token, Embedding, Transformer, Attention, Parameter,
Fine-tuning, RAG, MCP 같은 용어를 계속 만나게 됩니다.

각각의 개념을 따로 공부하면 이해할 수 있지만, 처음에는 이 기술들이 서로
어떻게 연결되는지 알기 어렵습니다.

LLM의 전체 구조는 크게 보면 생각보다 단순합니다.

``` text
대량의 Text
   ↓
Tokenization
   ↓
Embedding
   ↓
Transformer
   ↓
다음 Token 예측
   ↓
Loss
   ↓
Backpropagation
   ↓
Parameter 학습
   ↓
LLM
```

학습이 끝난 모델을 사용할 때는 다음과 같습니다.

``` text
사용자 Prompt
   ↓
Tokenization
   ↓
Embedding
   ↓
Transformer
   ↓
다음 Token 확률 계산
   ↓
Token 선택
   ↓
문장에 추가
   ↓
다시 다음 Token 계산
   ↓
반복
```

그리고 실제 서비스에서는 여기에 RAG, Tool Calling, MCP, Agent 같은
기술을 연결합니다.

이 글에서는 개별 기술을 깊게 설명하기보다 **LLM이 만들어지고 답변을
생성하며 외부 시스템과 연결되는 전체 흐름**을 한 번에 정리합니다.

> 이 글은 LLM의 전체 지도를 먼저 이해하기 위한 요약 글입니다. 각 기술의
> 세부 원리는 별도의 글에서 자세히 다룹니다. {: .notice--info}

------------------------------------------------------------------------

# 먼저 전체 구조부터 보기

LLM을 이해할 때는 다음 네 단계로 나누면 쉽습니다.

``` text
1. 데이터를 숫자로 바꾼다
        ↓
2. Transformer로 관계를 계산한다
        ↓
3. 다음 Token을 예측하도록 학습한다
        ↓
4. 학습된 모델로 Token을 반복 생성한다
```

실제 서비스를 만들 때는 여기에 다음 기술들이 추가됩니다.

``` text
                    LLM
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
       RAG       Fine-tuning     Tool
        │            │            │
   외부 지식      행동 조정     외부 기능 실행
                                  │
                                 MCP
                                  │
                                Agent
```

------------------------------------------------------------------------

# 1. LLM은 문장을 그대로 읽지 않는다

컴퓨터는 우리가 보는 문장을 그대로 이해하지 않습니다.

먼저 문장을 **Token**이라는 작은 단위로 나눕니다.

예를 들어

``` text
대한민국의 수도는 서울입니다.
```

라는 문장이 모델에 따라 개념적으로

``` text
대한민국
의
수도
는
서울
입니다
.
```

처럼 나뉠 수 있습니다.

각 Token은 숫자인 **Token ID**로 변환됩니다.

``` text
"대한민국" → 18231
"수도"     → 7312
"서울"     → 4921
```

이 과정을 **Tokenization**이라고 합니다.

Token ID는 단순한 식별 번호이므로 숫자의 크기 자체에 의미가 있는 것은
아닙니다.

------------------------------------------------------------------------

# 2. Token을 Vector로 바꾼다

Token ID를 신경망이 계산하기 좋은 형태로 바꾸는 과정이
**Embedding**입니다.

``` text
Token
 ↓
Token ID
 ↓
Embedding
 ↓
Vector
```

예를 들어 실제 차원을 크게 줄여 표현하면

``` text
서울
↓
[0.21, -0.72, 0.38, 0.91, ...]

부산
↓
[0.19, -0.68, 0.41, 0.87, ...]
```

와 같은 형태가 됩니다.

LLM 내부에서는 단어를 문자열로 처리하는 것이 아니라 이런 **고차원
Vector**를 대상으로 계산합니다.

Embedding도 학습되는 Parameter의 일부입니다.

------------------------------------------------------------------------

# 3. Transformer가 문맥을 계산한다

현대 LLM의 핵심 신경망 구조가 **Transformer**입니다.

Transformer에서 가장 중요한 개념은 **Attention**입니다.

다음 문장을 생각해보겠습니다.

``` text
철수는 학교에 갔다. 그는 친구를 만났다.
```

`그는`이라는 표현을 처리할 때 앞에 등장한 `철수`와의 관계가 중요합니다.

Transformer는 각 Token이 다른 Token을 얼마나 참고해야 하는지를
계산합니다.

``` text
철수는 ─────────┐
학교에           │
갔다             ├──→ 그는
                 │
친구를           │
만났다 ──────────┘
```

이 관계를 계산하는 것이 **Self-Attention**입니다.

------------------------------------------------------------------------

# Query, Key, Value

Attention에서는 각 Token의 표현을 이용해 세 종류의 Vector를 만듭니다.

``` text
Query
Key
Value
```

보통 다음처럼 표현합니다.

``` text
Q = XWq
K = XWk
V = XWv
```

간단하게 이해하면

``` text
Query
→ 지금 어떤 정보를 찾고 있는가

Key
→ 나는 어떤 정보와 관련되어 있는가

Value
→ 실제 전달할 정보
```

입니다.

Query와 Key를 비교하여 Token 사이의 관련도를 계산하고 그 중요도에 따라
Value를 가져옵니다.

Transformer의 Attention 공식은 다음과 같습니다.

``` text
                 QKᵀ
Attention = softmax(────)V
                  √dk
```

이 과정을 통해 각 Token은 주변 문맥을 반영한 새로운 Vector가 됩니다.

------------------------------------------------------------------------

# Transformer Block

Transformer는 Attention만으로 구성되어 있지 않습니다.

단순화하면 하나의 Transformer Block에는 다음 요소가 있습니다.

``` text
Input
 ↓
Self-Attention
 ↓
Residual / Normalization
 ↓
Feed Forward Network
 ↓
Residual / Normalization
 ↓
Output
```

이 Block을 여러 층 쌓습니다.

``` text
Embedding
 ↓
Transformer Block 1
 ↓
Transformer Block 2
 ↓
Transformer Block 3
 ↓
...
 ↓
Transformer Block N
 ↓
Output
```

GPT와 같은 현대 LLM은 이러한 Transformer를 매우 큰 규모로 구성한 언어
모델이라고 이해할 수 있습니다.

------------------------------------------------------------------------

# 4. Parameter는 AI가 학습하는 숫자다

Transformer 안에는 엄청나게 많은 Weight가 존재합니다.

예를 들어 Attention에는

``` text
Wq
Wk
Wv
```

가 있고 Feed Forward Network에도 여러 Weight Matrix가 있습니다.

Embedding Matrix 역시 Parameter입니다.

``` text
LLM Parameter

├─ Embedding
├─ Attention Weight
│   ├─ Wq
│   ├─ Wk
│   ├─ Wv
│   └─ Output Weight
├─ Feed Forward Weight
└─ 기타 Parameter
```

모델이 `7B Parameters`라고 한다면 약 70억 개 규모의 학습 가능한
Parameter를 가진다는 의미입니다.

AI가 학습한다는 것은 결국 이런 **Parameter를 데이터에 맞게 수정하는
과정**입니다.

------------------------------------------------------------------------

# 5. LLM은 다음 Token을 예측하도록 학습한다

LLM의 기본적인 학습 목표는 매우 단순합니다.

> 지금까지의 Token을 보고 다음 Token을 예측한다.

예를 들어

``` text
대한민국의 수도는
```

이라는 입력이 있다면 정답 Token이

``` text
서울
```

이 되도록 학습할 수 있습니다.

모델이 처음에는 다음처럼 예측할 수도 있습니다.

``` text
서울  10%
부산  50%
대전  20%
기타  20%
```

정답은 서울인데 모델은 부산의 확률을 더 높게 주었습니다.

이때 **Loss Function**을 이용해 얼마나 잘못 예측했는지를 계산합니다.

------------------------------------------------------------------------

# 6. Loss와 Backpropagation으로 학습한다

신경망 학습의 전체 과정은 다음과 같습니다.

``` text
학습 데이터
   ↓
Forward
   ↓
Prediction
   ↓
정답과 비교
   ↓
Loss
   ↓
Backpropagation
   ↓
Gradient
   ↓
Optimizer
   ↓
Parameter Update
   ↓
다시 반복
```

**Loss**는 모델의 예측이 정답과 얼마나 다른지를 나타냅니다.

**Backpropagation**은 각각의 Parameter가 Loss에 얼마나 영향을 주었는지
계산하는 과정입니다.

**Gradient**는 Parameter를 어느 방향으로 바꾸면 Loss가 줄어드는지를
알려줍니다.

Optimizer는 이 Gradient를 이용해 Weight를 수정합니다.

``` text
Prediction
 ↓
틀림
 ↓
Loss 계산
 ↓
어떤 Weight가 영향을 줬는지 계산
 ↓
Weight 수정
 ↓
다시 Prediction
```

이 과정을 엄청난 양의 데이터에 대해 반복하면서 LLM이 만들어집니다.

------------------------------------------------------------------------

# 7. Pre-training으로 기본 LLM을 만든다

대규모 텍스트 데이터를 이용해 기본적인 언어 능력을 학습하는 과정을
**Pre-training, 사전학습**이라고 합니다.

``` text
인터넷 문서
책
논문
코드
기타 Text
   ↓
데이터 정제
   ↓
Tokenization
   ↓
Transformer 학습
   ↓
Next Token Prediction
   ↓
Base Model
```

이 과정에서 모델은 사람이 문법과 지식을 하나씩 프로그래밍해서 넣는 것이
아니라 다음 Token을 더 잘 예측하는 방향으로 Parameter를 조정합니다.

그 결과 언어의 패턴, 개념 사이의 관계와 데이터에 포함된 다양한 지식이
모델의 Parameter에 분산된 형태로 반영됩니다.

------------------------------------------------------------------------

# 8. Base Model을 대화형 AI로 만든다

Pre-training이 끝났다고 바로 ChatGPT 같은 대화형 AI가 되는 것은
아닙니다.

Base Model은 기본적으로 다음 Token을 이어 쓰는 능력을 학습한 모델입니다.

이후 원하는 지시를 따르도록 추가적인 학습을 할 수 있습니다.

``` text
Pre-training
     ↓
Base Model
     ↓
Instruction / SFT
     ↓
Preference Training
     ↓
Chat Model
```

**SFT(Supervised Fine-Tuning)**에서는 질문과 원하는 답변 같은 예제를
이용해 모델이 지시를 따르는 방식을 학습시킵니다.

``` text
입력
"다음 문장을 요약해줘"

정답
"요약 결과..."
```

이런 데이터를 반복적으로 학습하면서 모델이 사용자의 지시 형식을 더 잘
따르도록 조정할 수 있습니다.

------------------------------------------------------------------------

# 9. Preference Training

답변에는 하나의 정확한 정답만 존재하지 않는 경우가 많습니다.

예를 들어 같은 질문에 대해

``` text
답변 A
→ 정확하고 간결함

답변 B
→ 장황하고 핵심을 놓침
```

이라면 사람이 A를 더 좋은 답변으로 평가할 수 있습니다.

이러한 **선호 정보**를 이용해 모델의 응답 성향을 조정할 수 있습니다.

``` text
Base Model
 ↓
SFT
 ↓
Preference Training
 ↓
사용자에게 더 적합한 Chat Model
```

구체적인 학습 방법은 모델과 개발 방식에 따라 다르지만 핵심은 단순한 다음
Token 예측으로 만든 Base Model을 실제 사용 목적에 맞게 추가 조정한다는
것입니다.

------------------------------------------------------------------------

# 10. 학습과 추론은 다르다

AI에서 반드시 구분해야 하는 것이 **Training과 Inference**입니다.

## Training

``` text
Input
 ↓
Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Weight Update
```

Parameter가 변경됩니다.

## Inference

``` text
Input
 ↓
Transformer
 ↓
Output
```

이미 학습된 Parameter를 사용해 결과를 계산하며 일반적인 사용 과정에서는
Weight를 변경하지 않습니다.

즉

``` text
Training
→ 모델을 만드는 과정

Inference
→ 만들어진 모델을 사용하는 과정
```

이라고 이해하면 됩니다.

------------------------------------------------------------------------

# 11. 사용자가 Prompt를 입력하면

이제 실제 Chat LLM을 사용한다고 생각해보겠습니다.

사용자가

``` text
대한민국의 수도는 어디야?
```

라고 입력합니다.

먼저 Prompt를 Token으로 변환합니다.

``` text
Prompt
 ↓
Tokenization
 ↓
Token IDs
 ↓
Embedding
```

그리고 Transformer를 통과합니다.

``` text
Embedding
 ↓
Transformer
 ↓
Transformer
 ↓
...
 ↓
Output
```

마지막에는 Vocabulary에 있는 각 Token 후보에 대한 **Logit**이
만들어집니다.

------------------------------------------------------------------------

# 12. Logit에서 다음 Token을 선택한다

Transformer의 마지막 출력에서는 다음 Token 후보마다 점수가 만들어집니다.

``` text
서울    12.5
부산     7.1
대한민국  5.2
도쿄     1.8
...
```

이 값을 **Logit**이라고 합니다.

Softmax 등을 적용하면 확률 분포로 바꿀 수 있습니다.

``` text
서울    92%
부산     4%
대한민국  3%
도쿄     1%
```

그리고 Decoding 방법에 따라 다음 Token을 선택합니다.

``` text
서울
```

Temperature, Top-k, Top-p 같은 설정은 이 Token 선택 과정에 영향을 줄 수
있습니다.

------------------------------------------------------------------------

# 13. LLM은 답변 전체를 한 번에 만드는 것이 아니다

일반적인 Autoregressive LLM은 답변 전체를 한 번에 생성하지 않습니다.

Token을 하나 생성하고 다시 다음 Token을 계산합니다.

``` text
Prompt
 ↓
Token 1
 ↓
Prompt + Token 1
 ↓
Token 2
 ↓
Prompt + Token 1 + Token 2
 ↓
Token 3
 ↓
...
```

예를 들어

``` text
대한민국의 수도는
```

에서

``` text
서울
```

을 생성했다면 다음 계산에서는

``` text
대한민국의 수도는 서울
```

이라는 문맥을 기반으로 다음 Token을 예측합니다.

이 과정을 종료 조건을 만날 때까지 반복하면 하나의 답변이 만들어집니다.

------------------------------------------------------------------------

# 14. Context Window

LLM이 현재 추론에서 참고할 수 있는 Token 범위를 **Context Window**라고
합니다.

Context에는 모델과 서비스 구조에 따라 다음과 같은 정보가 들어갈 수
있습니다.

``` text
System Prompt
사용자 질문
이전 대화
첨부된 정보
RAG 검색 결과
Tool 실행 결과
```

중요한 것은 **Context와 모델의 Weight는 서로 다르다**는 것입니다.

``` text
Weight
→ 학습 과정에서 만들어진 Parameter

Context
→ 현재 요청에서 모델에게 제공된 정보
```

대화를 길게 이어가거나 문서를 Prompt에 넣는다고 해서 그 내용이 자동으로
모델의 Weight에 학습되는 것은 아닙니다.

------------------------------------------------------------------------

# 15. KV Cache

LLM은 Token을 하나씩 생성하기 때문에 이전 Token의 계산을 매번 처음부터
반복하면 매우 비효율적입니다.

그래서 이전 Token에서 계산한 Key와 Value를 저장해 재사용합니다.

이를 **KV Cache**라고 합니다.

``` text
Prompt 처리
 ↓
K / V 계산
 ↓
KV Cache 저장
 ↓
새 Token 생성
 ↓
기존 KV Cache 재사용
 ↓
새 K / V 추가
 ↓
다음 Token 생성
```

Context가 길어질수록 KV Cache가 사용하는 메모리도 증가할 수 있습니다.

------------------------------------------------------------------------

# 16. LLM이 항상 사실을 말하는 것은 아니다

LLM의 기본 학습 목표를 다시 생각해보겠습니다.

``` text
주어진 문맥
 ↓
다음 Token 예측
```

모델의 기본 목적은 데이터베이스에서 사실을 검색해 검증하는 것이
아닙니다.

따라서

``` text
자연스러운 문장
≠
항상 사실인 문장
```

입니다.

모델이 사실처럼 보이지만 잘못된 내용을 생성하는 문제를 일반적으로
**Hallucination**이라고 부릅니다.

이 문제를 완전히 없애기는 어렵기 때문에 정확한 외부 정보가 필요한 경우
RAG나 Tool을 사용합니다.

------------------------------------------------------------------------

# 17. RAG는 외부 지식을 넣는다

LLM이 회사 내부 문서나 최신 정보를 알아야 한다고 해보겠습니다.

모델을 매번 다시 학습하는 대신 필요한 정보를 검색해 Context에 넣을 수
있습니다.

이것이 **RAG(Retrieval-Augmented Generation)**입니다.

``` text
사용자 질문
 ↓
검색
 ↓
관련 문서
 ↓
Context에 추가
 ↓
LLM
 ↓
답변
```

RAG의 중요한 특징은 일반적으로 **모델 Weight를 변경하지 않는다는
것**입니다.

``` text
RAG
→ Context를 변경

Fine-tuning
→ Parameter 또는 Adapter를 변경
```

이라고 구분하면 이해하기 쉽습니다.

------------------------------------------------------------------------

# 18. Fine-tuning은 모델의 행동을 조정한다

Fine-tuning은 이미 학습된 모델을 추가 데이터로 다시 학습하는 과정입니다.

``` text
Pre-trained Model
       +
Fine-tuning Data
       ↓
추가 학습
       ↓
조정된 Model
```

예를 들어 특정 형식의 답변, 조직 고유의 응답 패턴, 분류나 추출 작업 등을
반복적으로 수행하도록 조정할 수 있습니다.

반면 최신 회사 규정을 정확히 조회하는 문제라면 Fine-tuning보다 RAG나
데이터 조회 Tool이 더 적합할 수 있습니다.

간단하게 구분하면

``` text
최신 지식이 필요
→ RAG / Tool

특정 행동과 형식을 학습
→ Fine-tuning
```

으로 시작할 수 있습니다.

------------------------------------------------------------------------

# 19. LoRA와 QLoRA

큰 LLM의 모든 Parameter를 Fine-tuning하면 많은 GPU 메모리와 계산 자원이
필요합니다.

**LoRA**는 원본 Weight 대부분을 고정하고 작은 추가 행렬을 학습하는
방법입니다.

``` text
Original Weight
     ↓
    고정

LoRA Adapter
     ↓
    학습
```

이를 통해 전체 Parameter를 모두 수정하는 것보다 적은 학습 Parameter로
모델을 조정할 수 있습니다.

**QLoRA**는 양자화된 기본 모델을 활용해 LoRA 학습의 메모리 요구량을 더
줄이는 접근입니다.

------------------------------------------------------------------------

# 20. Tool Calling

LLM 자체는 기본적으로 Text를 입력받아 다음 Token을 생성하는 모델입니다.

하지만 실제 서비스에서는 외부 기능을 실행해야 할 수 있습니다.

예를 들어

``` text
"서울의 현재 날씨 알려줘"
```

라는 요청이 들어왔다고 해보겠습니다.

LLM이 학습된 기억만으로 현재 날씨를 말하는 것보다 실제 날씨 API를
호출하는 것이 정확합니다.

``` text
사용자
 ↓
LLM
 ↓
날씨 Tool 필요 판단
 ↓
weather(location="Seoul")
 ↓
외부 API 실행
 ↓
결과
 ↓
LLM
 ↓
최종 답변
```

이것이 **Tool Calling / Function Calling**의 기본 개념입니다.

------------------------------------------------------------------------

# 21. MCP

LLM에 연결되는 Tool이 많아지면 각각의 서비스마다 별도의 연결 방식을
만드는 것이 복잡해집니다.

**MCP(Model Context Protocol)**는 AI 애플리케이션과 외부 도구·데이터
소스를 연결하기 위한 표준화된 인터페이스를 제공합니다.

개념적으로

``` text
LLM Application
      ↓
     MCP
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
File  DB   Tool
```

처럼 이해할 수 있습니다.

중요한 것은 MCP 자체가 AI 모델은 아니라는 것입니다.

Transformer가 **LLM 내부 계산 구조**라면 MCP는 **LLM 애플리케이션과 외부
시스템을 연결하는 계층**입니다.

------------------------------------------------------------------------

# 22. Agent

Tool을 사용할 수 있는 LLM이 목표를 달성하기 위해 여러 단계를 판단하고
실행하도록 만들면 **AI Agent** 구조로 확장할 수 있습니다.

``` text
사용자 목표
   ↓
LLM
   ↓
현재 상황 판단
   ↓
Tool 선택
   ↓
Tool 실행
   ↓
결과 확인
   ↓
다음 행동 판단
   ↓
...
   ↓
목표 완료
```

단순 Chatbot이 한 번 질문을 받고 답하는 것과 달리 Agent는 여러 단계의
작업을 수행할 수 있습니다.

예를 들어

``` text
"이번 주 매출 보고서를 만들어줘"
```

라는 요청에서 Agent가

``` text
DB 조회
 ↓
매출 데이터 분석
 ↓
지난주 데이터 비교
 ↓
차트 생성
 ↓
문서 작성
```

처럼 여러 Tool을 순서대로 사용할 수 있습니다.

------------------------------------------------------------------------

# 23. LangChain과 LangGraph는 어디에 위치하는가

LangChain이나 LangGraph는 LLM 자체가 아닙니다.

LLM을 실제 애플리케이션으로 구성할 때 사용하는 프레임워크입니다.

``` text
                AI Application
                       │
          LangChain / LangGraph
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
         LLM          RAG          Tool
                                      ↓
                                     MCP
```

LangChain은 LLM, Prompt, Retriever, Tool 등을 연결하는 여러 기능을
제공하고 LangGraph는 상태를 가진 복잡한 Workflow나 Agent 흐름을 Graph
형태로 구성하는 데 사용할 수 있습니다.

------------------------------------------------------------------------

# 24. 양자화는 모델을 작게 만든다

LLM의 Parameter는 숫자로 저장됩니다.

높은 정밀도의 숫자를 사용하면 많은 메모리가 필요합니다.

예를 들어 매우 단순하게 Weight만 생각하면 70억 Parameter 모델을 FP16으로
저장할 경우

``` text
7B × 2 Byte
≈ 14 GB
```

정도의 공간이 필요합니다.

실제 실행 메모리는 KV Cache, Activation, Runtime Overhead 등에 따라 더
필요할 수 있습니다.

**Quantization, 양자화**는 Weight 등을 더 낮은 비트 수로 표현해 모델의
저장 공간과 메모리 요구량을 줄이는 기술입니다.

``` text
FP32
 ↓
FP16 / BF16
 ↓
INT8
 ↓
4-bit
```

낮은 정밀도를 사용할수록 메모리를 줄일 수 있지만 품질과 성능의
Trade-off가 생길 수 있습니다.

------------------------------------------------------------------------

# 25. Serving

학습된 LLM을 실제 사용자가 호출할 수 있도록 서버에서 실행하는 것을
**Serving**이라고 합니다.

``` text
사용자
 ↓
API
 ↓
LLM Serving Server
 ↓
GPU
 ↓
Model
 ↓
Response
```

대표적인 로컬·서버용 실행 도구와 프레임워크로 Ollama, vLLM, SGLang 등이
있습니다.

서비스 환경에서는 단순히 모델을 실행하는 것뿐 아니라

``` text
동시 요청
Batching
KV Cache 관리
GPU Memory
Token 생성 속도
Latency
Throughput
```

등도 중요합니다.

------------------------------------------------------------------------

# LLM 전체 구조를 한 장으로 정리하면

지금까지의 내용을 하나로 연결하면 다음과 같습니다.

``` text
                     [학습]

대규모 Text Data
       ↓
  Tokenization
       ↓
    Embedding
       ↓
  Transformer
       ↓
Next Token Prediction
       ↓
      Loss
       ↓
 Backpropagation
       ↓
   Optimizer
       ↓
Parameter Update
       ↓
  반복 학습
       ↓
   Base LLM
       ↓
SFT / Preference Training
       ↓
    Chat LLM


                     [추론]

사용자 Prompt
       ↓
  Tokenization
       ↓
    Embedding
       ↓
  Transformer
       ↓
     Logits
       ↓
다음 Token 선택
       ↓
 Context에 추가
       ↓
다음 Token 다시 계산
       ↓
      반복
       ↓
     답변


                 [외부 확장]

                    LLM
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
      RAG       Fine-tuning       Tool
       │             │             │
외부 지식 제공   Parameter 조정   외부 기능 실행
                                     │
                                    MCP
                                     │
                                   Agent


                 [실제 서비스]

                    LLM
                     ↓
              Quantization
                     ↓
                 Serving
                     ↓
                   GPU
                     ↓
                   API
                     ↓
                  사용자
```

------------------------------------------------------------------------

# 각 기술은 어디에 속하는가

  기술                    역할
  ----------------------- ------------------------------------------------
  Tokenization            Text를 Token으로 분리하고 ID로 변환
  Embedding               Token을 Vector로 변환
  Neural Network          데이터를 이용해 Parameter를 학습하는 기반 구조
  Transformer             Attention을 이용해 Token 관계를 계산
  Attention               어떤 Token을 얼마나 참고할지 계산
  Parameter               학습으로 변경되는 모델 내부 숫자
  Loss                    모델이 얼마나 틀렸는지 계산
  Backpropagation         Parameter별 Gradient 계산
  Pre-training            대규모 데이터로 기본 언어 모델 학습
  SFT                     원하는 지시와 응답 방식을 추가 학습
  Preference Training     더 선호되는 응답 방향으로 모델 조정
  Context Window          현재 추론에서 참고할 수 있는 Token 범위
  KV Cache                이전 Token의 K/V를 저장해 생성 계산을 효율화
  RAG                     외부 정보를 검색해 Context에 추가
  Fine-tuning             모델 Parameter 또는 Adapter를 추가 학습
  LoRA                    적은 추가 Parameter로 Fine-tuning
  Tool Calling            LLM이 외부 함수나 서비스를 사용
  MCP                     AI 애플리케이션과 외부 Tool·Data 연결을 표준화
  Agent                   LLM이 Tool을 이용해 여러 단계의 작업 수행
  LangChain / LangGraph   LLM 애플리케이션과 Workflow 구성
  Quantization            모델의 메모리 요구량을 줄임
  Serving                 학습된 모델을 실제 서비스에서 실행

------------------------------------------------------------------------

# 처음 공부한다면 어떤 순서가 좋은가

LLM을 처음부터 이해하려면 개별 기술을 무작위로 공부하기보다 다음 순서가
좋습니다.

``` text
1. 신경망 학습 원리
        ↓
2. Token과 Embedding
        ↓
3. Transformer
        ↓
4. LLM의 학습과 추론
        ↓
5. Context와 KV Cache
        ↓
6. RAG와 Fine-tuning
        ↓
7. Tool Calling과 MCP
        ↓
8. Agent / LangChain / LangGraph
        ↓
9. Quantization과 Serving
```

가장 중요한 부분은 앞쪽입니다.

``` text
신경망
  ↓
Token / Embedding
  ↓
Transformer
  ↓
Next Token Prediction
```

이 부분을 이해하면 LLM 자체가 어떻게 동작하는지 이해할 수 있습니다.

뒤쪽의

``` text
RAG
Fine-tuning
MCP
Agent
Serving
```

은 만들어진 LLM을 **어떻게 확장하고 실제 시스템에서 사용할 것인가**에
가까운 영역입니다.

------------------------------------------------------------------------

# 가장 중요한 개념만 다시 정리하면

LLM은 마법처럼 문장을 이해해서 답을 꺼내는 프로그램이 아닙니다.

먼저 Text를 Token으로 나누고 Vector로 변환합니다.

``` text
Text
 ↓
Token
 ↓
Embedding
```

Transformer는 Attention을 이용해 Token 사이의 문맥 관계를 계산합니다.

``` text
Embedding
 ↓
Transformer
 ↓
Context를 반영한 표현
```

학습할 때는 다음 Token을 잘 맞히도록 Parameter를 반복적으로 수정합니다.

``` text
Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Parameter Update
```

사용할 때는 학습된 Parameter를 이용해 다음 Token을 반복적으로
생성합니다.

``` text
Prompt
 ↓
Next Token
 ↓
Next Token
 ↓
Next Token
 ↓
...
 ↓
Answer
```

LLM에 없는 최신 정보나 사내 정보를 제공하려면 RAG를 사용할 수 있고,
모델의 행동이나 출력 패턴을 바꾸려면 Fine-tuning을 사용할 수 있습니다.

외부 프로그램을 실행하려면 Tool Calling을 사용하고, 다양한 Tool과
데이터를 표준화해 연결하는 방법으로 MCP를 사용할 수 있습니다.

LLM이 여러 Tool을 사용하면서 여러 단계의 작업을 수행하도록 구성하면
Agent로 확장됩니다.

결국 전체 구조는 다음과 같이 기억하면 됩니다.

``` text
                AI의 기반
                   │
                신경망
                   ↓
           Token / Embedding
                   ↓
              Transformer
                   ↓
                  LLM
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
      RAG      Fine-tuning    Tool
                               ↓
                              MCP
                               ↓
                             Agent
                               ↓
                            Serving
```

------------------------------------------------------------------------

# 정리

LLM을 이해하기 위해 모든 수학식과 구현 코드를 먼저 알 필요는 없습니다.

우선 다음 흐름을 이해하는 것이 중요합니다.

> **Text를 Token으로 나누고 Embedding Vector로 변환한 뒤 Transformer가
> 문맥 관계를 계산한다. 모델은 다음 Token을 잘 예측하도록 Loss와
> Backpropagation을 이용해 Parameter를 학습하며, 추론할 때는 학습된
> Parameter를 이용해 다음 Token을 반복적으로 생성한다.**

그리고 실제 AI 서비스에서는 이 LLM을 중심으로 다른 기술이 연결됩니다.

``` text
RAG
→ 필요한 외부 지식을 제공

Fine-tuning
→ 모델의 행동과 패턴을 추가 학습

Tool Calling
→ 외부 기능 실행

MCP
→ Tool과 Data 연결 표준화

Agent
→ 여러 단계를 판단하고 실행

Quantization
→ 모델을 더 작게 실행

Serving
→ 실제 사용자에게 모델 제공
```

각각의 기술은 서로 완전히 별개의 AI가 아니라 **LLM을 만들고, 사용하고,
확장하고, 서비스하기 위한 서로 다른 계층의 기술**입니다.

이 전체 지도를 먼저 머릿속에 넣은 뒤 Token, 신경망, Transformer, RAG,
Fine-tuning, MCP 같은 개별 주제를 하나씩 자세히 공부하면 각 기술이 왜
필요한지 훨씬 쉽게 이해할 수 있습니다.

## 참고 자료

-   Ashish Vaswani 외, [Attention Is All You
    Need](https://arxiv.org/abs/1706.03762)
-   Ian Goodfellow, Yoshua Bengio, Aaron Courville, [Deep
    Learning](https://www.deeplearningbook.org/)
-   Tom B. Brown 외, [Language Models are Few-Shot
    Learners](https://arxiv.org/abs/2005.14165)
-   Patrick Lewis 외, [Retrieval-Augmented Generation for
    Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
-   Edward Hu 외, [LoRA: Low-Rank Adaptation of Large Language
    Models](https://arxiv.org/abs/2106.09685)
-   Tim Dettmers 외, [QLoRA: Efficient Finetuning of Quantized
    LLMs](https://arxiv.org/abs/2305.14314)
-   Model Context Protocol, [공식
    문서](https://modelcontextprotocol.io/)
