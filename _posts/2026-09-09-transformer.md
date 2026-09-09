---
layout: single
title: "Transformer : Attention부터 LLM의 동작 원리까지"
categories:
  - "AI"
permalink: /topics/ai/transformer/
tags:
  - "AI"
  - "Transformer"
  - "Attention"
  - "Self-Attention"
  - "LLM"
  - "GPT"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

ChatGPT, GPT, Claude, Gemini, Llama와 같은 현대 LLM을 이해하려면 **Transformer**를 이해해야 합니다.

Transformer는 2017년 논문 **Attention Is All You Need**에서 제안된 신경망 구조입니다.

이전에도 자연어를 처리하는 신경망은 존재했지만 Transformer는 **Attention**을 중심으로 문장 전체의 관계를 처리할 수 있도록 설계되었습니다.

오늘날 대부분의 LLM은 Transformer 또는 Transformer에서 발전한 구조를 사용합니다.

이 글에서는 Transformer를 단순히 구조만 외우는 것이 아니라 다음 흐름으로 이해해보겠습니다.

~~~text
Text
 ↓
Tokenization
 ↓
Embedding
 ↓
Position Information
 ↓
Transformer Block
 ├─ Attention
 └─ Feed Forward Network
 ↓
다음 Token 확률
 ↓
Token 선택
 ↓
반복
~~~

---

# Transformer가 등장하기 전

Transformer 이전에는 자연어 처리에 **RNN(Recurrent Neural Network)**이나 **LSTM(Long Short-Term Memory)**이 많이 사용되었습니다.

RNN의 기본 아이디어는 문장을 순서대로 처리하는 것입니다.

~~~text
나는 → 오늘 → 학교에 → 갔다
~~~

첫 번째 단어를 처리한 결과를 다음 단계로 넘깁니다.

~~~text
나는
 ↓
상태 1
 ↓
오늘
 ↓
상태 2
 ↓
학교에
 ↓
상태 3
 ↓
갔다
~~~

이 방식은 문장의 순서를 자연스럽게 처리할 수 있다는 장점이 있습니다.

하지만 큰 문제가 있습니다.

---

# RNN의 문제

## 순차적으로 계산해야 한다

RNN은 이전 단계의 계산 결과가 있어야 다음 단계를 계산할 수 있습니다.

~~~text
Token 1
  ↓
Token 2
  ↓
Token 3
  ↓
Token 4
~~~

따라서 긴 문장을 처리할 때 병렬 계산이 어렵습니다.

GPU는 대량의 연산을 동시에 수행하는 데 강하지만 RNN 구조에서는 이러한 장점을 충분히 활용하기 어렵습니다.

---

## 먼 단어의 관계를 학습하기 어렵다

다음 문장을 생각해보겠습니다.

~~~text
철수는 학교에서 친구들과 오랫동안 이야기를 나눈 뒤 집으로 돌아갔다.
~~~

`철수`와 `돌아갔다`는 서로 멀리 떨어져 있지만 의미적으로 연결되어 있습니다.

문장이 길어질수록 초기 정보가 여러 단계를 거쳐 전달되어야 하기 때문에 장거리 관계를 학습하기 어려운 문제가 있었습니다.

LSTM과 GRU는 이러한 문제를 개선했지만 순차 계산이라는 근본적인 특징은 남아 있었습니다.

---

# Transformer의 핵심 아이디어

Transformer는 접근 방식을 바꿨습니다.

문장을 반드시 한 단어씩 순서대로 처리하지 않고 **문장 안의 Token들이 서로 어떤 관계를 가지고 있는지 직접 계산**합니다.

예를 들어

~~~text
철수는 학교에 갔다
~~~

라는 문장이 있다면 각 Token이 다른 Token과 얼마나 관련되어 있는지를 계산합니다.

~~~text
철수는 ───── 학교에
   │           │
   └───────── 갔다
~~~

이 관계를 계산하는 핵심 기술이 **Attention**입니다.

> Transformer의 핵심은 문장 속 각 Token이 다른 Token을 얼마나 참고해야 하는지를 계산하는 것이다.

---

# Transformer 전체 흐름

LLM 관점에서 Transformer의 흐름을 단순화하면 다음과 같습니다.

~~~text
"대한민국의 수도는"
        ↓
Tokenization
        ↓
Token ID
        ↓
Embedding
        ↓
Position Information
        ↓
Transformer Block
        ↓
Transformer Block
        ↓
Transformer Block
        ↓
...
        ↓
Logits
        ↓
Token Probability
        ↓
"서울"
~~~

각 단계를 하나씩 살펴보겠습니다.

---

# Tokenization

Transformer가 문장을 직접 이해하는 것은 아닙니다.

먼저 문장을 **Token**이라는 단위로 나눕니다.

예를 들어

~~~text
대한민국의 수도는 서울입니다.
~~~

가 실제 모델에서는 개념적으로 다음처럼 나뉠 수 있습니다.

~~~text
대한
민국
의
수도
는
서울
입니다
.
~~~

실제 Token 분할 방식은 사용하는 Tokenizer와 모델에 따라 다릅니다.

각 Token은 숫자인 **Token ID**로 변환됩니다.

~~~text
"대한" → 18342
"민국" → 9271
"수도" → 4512
"서울" → 7821
~~~

Transformer가 실제로 처리하는 것은 문자열 자체가 아니라 이런 숫자화된 입력입니다.

---

# Embedding

Token ID는 단순한 번호입니다.

~~~text
서울 = 7821
부산 = 3921
한국 = 812
~~~

숫자 자체의 크기에는 의미가 없습니다.

`7821`이 `3921`보다 의미적으로 크다는 뜻이 아닙니다.

따라서 Token ID를 신경망이 계산하기 좋은 **Vector**로 변환합니다.

이것이 **Embedding**입니다.

~~~text
Token ID
   ↓
Embedding Table
   ↓
Vector
~~~

예를 들어 실제 차원 수를 크게 단순화하면

~~~text
서울
↓
[0.21, -0.73, 0.18, 0.91, ...]

부산
↓
[0.19, -0.69, 0.22, 0.87, ...]
~~~

와 같은 형태입니다.

실제 LLM의 Embedding은 수백~수천 차원 이상의 벡터를 사용할 수 있습니다.

Transformer 내부의 대부분 계산은 이러한 벡터와 행렬을 대상으로 이루어집니다.

---

# Embedding도 학습된다

Embedding은 사람이 직접 숫자를 입력해서 만드는 것이 아닙니다.

Embedding Matrix 역시 모델의 **Parameter**입니다.

~~~text
Token
 ↓
Embedding Vector
 ↓
Transformer
 ↓
Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Embedding Parameter 수정
~~~

학습 과정에서 다음 Token을 더 잘 예측하는 방향으로 Embedding도 함께 조정됩니다.

---

# 위치 정보가 필요한 이유

Transformer의 Attention 계산 자체는 Token의 순서를 자동으로 이해하지 못합니다.

다음 두 문장을 생각해보겠습니다.

~~~text
철수가 영희를 좋아한다.

영희가 철수를 좋아한다.
~~~

등장하는 단어가 비슷하더라도 순서에 따라 의미가 완전히 달라집니다.

따라서 Transformer에는 Token의 **위치 정보(Position Information)**가 필요합니다.

초기 Transformer에서는 **Positional Encoding**을 사용했습니다.

개념적으로

~~~text
Token Embedding
      +
Position Information
      ↓
Transformer
~~~

와 같은 구조입니다.

---

# Positional Encoding

원래 Transformer 논문에서는 sin과 cos 함수를 이용해 위치 정보를 만들었습니다.

개념적으로

~~~text
Token 1 → 위치 정보 1
Token 2 → 위치 정보 2
Token 3 → 위치 정보 3
...
~~~

을 Embedding에 추가합니다.

이를 통해 모델은

~~~text
어떤 Token인가?
+
문장의 몇 번째 위치인가?
~~~

를 함께 사용할 수 있습니다.

---

# RoPE

현대 LLM에서는 **RoPE(Rotary Positional Embedding)**를 사용하는 경우가 많습니다.

RoPE는 위치 정보를 단순히 Embedding에 더하는 대신 Query와 Key 벡터에 위치에 따른 회전 변환을 적용합니다.

이를 통해 Attention 계산에서 Token 사이의 상대적인 위치 관계를 표현할 수 있습니다.

~~~text
Token A 위치
      ↓
Query / Key 회전

Token B 위치
      ↓
Query / Key 회전

      ↓
Attention 계산에서
상대 위치 관계 반영
~~~

Transformer의 구현에 따라 위치 정보를 표현하는 방식은 달라질 수 있습니다.

---

# Attention이란 무엇인가

Transformer에서 가장 중요한 개념입니다.

문장을 보겠습니다.

~~~text
철수는 사과를 먹었다. 그는 배가 고팠다.
~~~

`그는`이라는 Token을 처리할 때 모델은 앞의 여러 Token 중 어떤 것을 중요하게 참고해야 할까요?

~~~text
철수는 ← 중요
사과를
먹었다
그는
~~~

이처럼 현재 Token을 처리할 때 다른 Token들이 얼마나 중요한지를 계산하는 것이 **Attention**입니다.

---

# Self-Attention

하나의 문장 안에서 Token들이 서로를 참고하는 것을 **Self-Attention**이라고 합니다.

예를 들어

~~~text
The cat sat on the mat
~~~

이라는 문장이 있다면 각 Token이 다른 모든 Token과 관계를 계산합니다.

~~~text
The ───────┐
cat ───────┤
sat ───────┤
on  ───────┤ → 서로 관계 계산
the ───────┤
mat ───────┘
~~~

즉 Token 하나를 독립적으로 처리하는 것이 아니라 **문맥 속에서 다른 Token과의 관계를 이용해 새로운 표현을 만드는 것**입니다.

---

# Query, Key, Value

Self-Attention을 이해하려면 **Query, Key, Value**를 알아야 합니다.

보통 다음과 같이 줄여서 표현합니다.

~~~text
Query = Q
Key   = K
Value = V
~~~

Embedding 또는 이전 Layer의 Hidden State를 각각 다른 Weight Matrix와 곱해 Q, K, V를 만듭니다.

~~~text
Q = XWq
K = XWk
V = XWv
~~~

여기서

~~~text
X
→ 현재 Token 표현

Wq
Wk
Wv
→ 학습되는 Weight Matrix
~~~

입니다.

즉 Q, K, V는 서로 완전히 다른 원본 데이터가 아니라 **같은 Token 표현을 서로 다른 목적의 공간으로 변환한 결과**입니다.

---

# Query의 의미

Query는 쉽게 말하면

> 내가 지금 어떤 정보를 찾고 있는가?

에 해당합니다.

~~~text
현재 Token
   ↓
Query
   ↓
"어떤 Token을 참고해야 하지?"
~~~

---

# Key의 의미

Key는

> 나는 어떤 정보와 관련되어 있는가?

를 판단하는 데 사용되는 값이라고 이해할 수 있습니다.

Query와 Key를 비교하여 두 Token의 관련도를 계산합니다.

~~~text
Query(Token A)
      ↕ 비교
Key(Token B)
~~~

---

# Value의 의미

Value는 실제로 전달할 정보입니다.

Query와 Key를 이용해 어떤 Token을 얼마나 참고할지 결정한 다음 해당 Token의 Value를 가중합합니다.

~~~text
Query + Key
    ↓
중요도 계산
    ↓
Value를 중요도만큼 가져옴
~~~

따라서 아주 단순하게 기억하면

~~~text
Query
→ 무엇을 찾는가

Key
→ 무엇과 관련 있는가

Value
→ 실제 전달할 정보
~~~

입니다.

---

# Attention Score 계산

Query와 Key가 얼마나 비슷한지를 계산하기 위해 **Dot Product**를 사용합니다.

~~~text
Score = Q · K
~~~

벡터의 방향이 비슷할수록 큰 값이 나올 수 있습니다.

예를 들어

~~~text
현재 Token의 Query
        ↓
각 Token의 Key와 비교

Q · K1
Q · K2
Q · K3
Q · K4
~~~

를 계산합니다.

결과는 각 Token을 얼마나 참고할지 결정하는 원재료가 됩니다.

---

# 왜 √dk로 나누는가

Transformer의 Attention 공식은 다음과 같습니다.

~~~text
Attention(Q, K, V)

          QKᵀ
= softmax(──────) V
           √dk
~~~

Q와 K의 차원이 커지면 Dot Product 값도 커질 가능성이 있습니다.

값이 너무 커지면 Softmax 결과가 지나치게 극단적으로 변하고 학습이 불안정해질 수 있습니다.

따라서

~~~text
√dk
~~~

로 나누어 값의 크기를 조절합니다.

그래서 이 방식을 **Scaled Dot-Product Attention**이라고 합니다.

---

# Softmax

Attention Score는 그대로 사용하지 않고 **Softmax**를 적용합니다.

예를 들어 Score가

~~~text
철수  4.2
학교  1.1
밥    0.7
~~~

라면 Softmax를 거쳐 개념적으로

~~~text
철수  0.91
학교  0.06
밥    0.03
~~~

와 같이 합이 1인 가중치로 변환됩니다.

이 값은 각 Token을 얼마나 참고할지를 나타냅니다.

---

# Value를 가중합한다

Attention Weight가 만들어졌다면 Value에 적용합니다.

~~~text
0.91 × V(철수)
+
0.06 × V(학교)
+
0.03 × V(밥)
~~~

결과적으로 현재 Token은 다른 Token의 정보를 중요도에 따라 섞은 새로운 Vector를 얻게 됩니다.

전체 과정을 정리하면

~~~text
Input
 ↓
Q, K, V 생성
 ↓
Q와 K 비교
 ↓
Attention Score
 ↓
Scale
 ↓
Softmax
 ↓
Attention Weight
 ↓
Value 가중합
 ↓
새로운 Token 표현
~~~

입니다.

---

# Attention Matrix

문장에 Token이 여러 개 있다면 각 Token이 다른 Token을 얼마나 참고하는지를 행렬 형태로 표현할 수 있습니다.

예를 들어

~~~text
          철수는  학교에  갔다
철수는      0.5    0.2    0.3
학교에      0.2    0.5    0.3
갔다        0.4    0.4    0.2
~~~

와 같은 형태입니다.

실제 값은 모델이 계산하며 위 숫자는 설명을 위한 예입니다.

Token 수가 `n`이라면 기본적인 Self-Attention에서는 대략 `n × n` 관계를 계산하게 됩니다.

이 때문에 문맥 길이가 길어질수록 Attention 계산량과 메모리 사용량이 크게 증가합니다.

---

# Multi-Head Attention

Attention을 한 번만 계산하면 하나의 관계 표현에 의존하게 됩니다.

Transformer는 여러 개의 Attention을 병렬로 계산합니다.

이를 **Multi-Head Attention**이라고 합니다.

~~~text
Input
 ├─ Head 1 → Attention
 ├─ Head 2 → Attention
 ├─ Head 3 → Attention
 └─ Head 4 → Attention
        ↓
      결합
        ↓
     Output
~~~

각 Head는 서로 다른 Wq, Wk, Wv를 학습할 수 있기 때문에 서로 다른 관계를 포착할 수 있습니다.

개념적으로 어떤 Head는

~~~text
주어 ↔ 동사
~~~

관계에 강하게 반응할 수 있고 다른 Head는

~~~text
대명사 ↔ 앞의 명사
~~~

또는 다른 형태의 문맥적 관계를 포착할 수 있습니다.

다만 특정 Head가 반드시 사람이 이해할 수 있는 하나의 문법 기능만 담당한다고 단정할 수는 없습니다.

---

# Feed Forward Network

Attention만으로 Transformer Block이 끝나는 것은 아닙니다.

Attention 결과는 **Feed Forward Network(FFN)**를 통과합니다.

~~~text
Attention
   ↓
Feed Forward Network
   ↓
Output
~~~

FFN은 각 Token 위치에 독립적으로 적용되는 작은 신경망이라고 볼 수 있습니다.

단순화하면

~~~text
x
 ↓
Linear
 ↓
Activation
 ↓
Linear
 ↓
Output
~~~

구조입니다.

수식으로 단순화하면

~~~text
FFN(x) = W2 × Activation(W1x + b1) + b2
~~~

와 같은 형태입니다.

현대 LLM에서는 GELU, SiLU, SwiGLU 등 다양한 활성화 및 FFN 변형을 사용합니다.

Attention이 **Token 사이의 관계를 섞는 역할**을 한다면 FFN은 각 Token의 표현을 **비선형적으로 변환하고 가공하는 역할**을 한다고 이해할 수 있습니다.

---

# Residual Connection

Transformer에는 **Residual Connection**도 사용됩니다.

어떤 Layer의 입력을 변환 결과에 다시 더합니다.

~~~text
Input ─────────────┐
  ↓                │
Attention          │
  ↓                │
Output ─────────── + 
                   ↓
                Result
~~~

수식으로는 개념적으로

~~~text
y = x + F(x)
~~~

입니다.

이렇게 하면 깊은 신경망에서 정보와 Gradient가 여러 Layer를 지나 전달되는 데 도움이 됩니다.

Transformer를 수십~수백 개 Layer로 깊게 쌓을 수 있는 데 중요한 요소 중 하나입니다.

---

# Layer Normalization

Transformer에서는 **Layer Normalization**도 중요한 역할을 합니다.

신경망 내부 값의 분포를 정규화하여 학습을 안정적으로 만드는 데 도움을 줍니다.

~~~text
Input
 ↓
Normalization
 ↓
Attention / FFN
~~~

원래 Transformer와 현대 LLM은 LayerNorm을 배치하는 방식도 조금씩 다릅니다.

대표적으로

~~~text
Post-LN
Pre-LN
~~~

구조가 있습니다.

현대 LLM에서는 LayerNorm 대신 **RMSNorm**을 사용하는 경우도 많습니다.

---

# Transformer Block

지금까지의 요소를 합치면 Transformer Block의 핵심 구조를 이해할 수 있습니다.

단순화하면

~~~text
Input
  │
  ├──────────────┐
  ↓              │
Normalization    │
  ↓              │
Multi-Head       │
Self-Attention   │
  ↓              │
  + ←────────────┘
  ↓
  ├──────────────┐
  ↓              │
Normalization    │
  ↓              │
Feed Forward     │
  ↓              │
  + ←────────────┘
  ↓
Output
~~~

실제 모델마다 세부 구조는 다르지만 핵심은

~~~text
Attention
+
Feed Forward
+
Residual Connection
+
Normalization
~~~

입니다.

---

# Transformer는 Block을 여러 번 쌓는다

Transformer 하나가 Attention을 한 번만 계산하는 것은 아닙니다.

Transformer Block을 여러 층 쌓습니다.

~~~text
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
~~~

각 Layer를 지나면서 Token 표현은 계속 변화합니다.

초기에는 Token 자체의 정보에 가까운 표현이었다면 여러 Layer를 거치면서 문맥을 반영한 표현으로 변합니다.

---

# Encoder와 Decoder

원래 Transformer는 크게 두 부분으로 구성되었습니다.

~~~text
Encoder
+
Decoder
~~~

전체 구조는 다음과 같습니다.

~~~text
Input Text
   ↓
Encoder
   ↓
Context Representation
   ↓
Decoder
   ↓
Output Text
~~~

---

# Encoder

Encoder는 입력 문장을 읽고 문맥을 반영한 표현을 만듭니다.

~~~text
Input
 ↓
Self-Attention
 ↓
Feed Forward
 ↓
Output Representation
~~~

Encoder의 Self-Attention은 일반적으로 입력 문장 전체를 서로 참고할 수 있습니다.

대표적인 Encoder 기반 모델로 **BERT** 계열이 있습니다.

Encoder 구조는 문장 분류, 정보 추출, 임베딩 등 입력을 이해하고 표현하는 작업에 적합합니다.

---

# Decoder

Decoder는 이전까지 생성된 Token을 기반으로 다음 Token을 생성합니다.

~~~text
Token 1
 ↓
Token 2
 ↓
Token 3
 ↓
Token 4
~~~

GPT 계열 LLM은 대표적인 **Decoder-only Transformer**입니다.

---

# Causal Mask

GPT가 다음 Token을 예측할 때 미래의 정답 Token을 미리 보면 안 됩니다.

예를 들어 학습 문장이

~~~text
나는 오늘 학교에 갔다
~~~

라고 해보겠습니다.

`오늘`을 처리하면서 미래에 있는 `학교에`, `갔다`를 미리 참고하면 다음 Token 예측 학습의 의미가 없어집니다.

따라서 미래 Token을 보지 못하도록 **Causal Mask**를 적용합니다.

~~~text
             참고 가능

나는       → 나는
오늘       → 나는 오늘
학교에     → 나는 오늘 학교에
갔다       → 나는 오늘 학교에 갔다
~~~

Attention Matrix로 표현하면 개념적으로

~~~text
          나는  오늘  학교에  갔다

나는       O     X      X      X
오늘       O     O      X      X
학교에     O     O      O      X
갔다       O     O      O      O
~~~

와 같습니다.

미래 위치의 Attention Score를 사실상 선택할 수 없도록 Mask 처리합니다.

이 때문에 GPT는 **Autoregressive Language Model**로 동작할 수 있습니다.

---

# Cross-Attention

원래 Encoder-Decoder Transformer에서는 Decoder가 Encoder의 결과를 참고해야 합니다.

이때 **Cross-Attention**을 사용합니다.

~~~text
Encoder Output
      ↓
   Key / Value
      ↑
Decoder Query
~~~

즉 Decoder가 현재 필요한 정보를 Query로 만들고 Encoder의 출력에서 관련 정보를 찾아오는 구조입니다.

번역을 예로 들면

~~~text
한국어 문장
   ↓
Encoder
   ↓
문맥 표현
   ↓
Cross-Attention
   ↓
Decoder
   ↓
영어 문장
~~~

처럼 동작할 수 있습니다.

---

# Transformer 모델의 세 가지 대표 구조

Transformer 기반 모델은 크게 세 형태로 구분할 수 있습니다.

| 구조 | 대표 모델 | 주요 용도 |
|---|---|---|
| Encoder-only | BERT | 이해, 분류, 임베딩 |
| Decoder-only | GPT, Llama | 텍스트 생성, LLM |
| Encoder-Decoder | T5 | 번역, 변환, 생성 |

---

# GPT는 왜 Decoder-only인가

GPT의 핵심 목적은 다음 Token을 계속 생성하는 것입니다.

~~~text
현재 문장
 ↓
다음 Token 예측
 ↓
문장에 추가
 ↓
다시 다음 Token 예측
~~~

따라서 Autoregressive 생성에 적합한 Decoder 구조를 사용합니다.

~~~text
Prompt
 ↓
Decoder Transformer
 ↓
Next Token
 ↓
Prompt + Next Token
 ↓
Decoder Transformer
 ↓
Next Token
 ↓
...
~~~

---

# GPT가 문장을 생성하는 과정

사용자가 다음과 같이 입력했다고 해보겠습니다.

~~~text
대한민국의 수도는
~~~

### 1. Tokenization

~~~text
대한민국
의
수도
는
~~~

실제 Token 분리는 모델마다 다릅니다.

### 2. Embedding

~~~text
Token
 ↓
Vector
~~~

각 Token을 벡터로 변환합니다.

### 3. Transformer

여러 Transformer Block을 통과합니다.

~~~text
Embedding
 ↓
Attention
 ↓
FFN
 ↓
Attention
 ↓
FFN
 ↓
...
~~~

### 4. Logits 출력

마지막 Hidden State를 Vocabulary 크기의 출력으로 변환하면 각 Token 후보에 대한 **Logit**이 만들어집니다.

개념적으로

~~~text
서울  12.4
부산   8.1
한국   6.3
도쿄   2.1
...
~~~

와 같은 값입니다.

### 5. 확률 계산

Logit에 Softmax 등을 적용하면 Token 후보의 확률 분포를 만들 수 있습니다.

~~~text
서울  0.91
부산  0.05
한국  0.03
도쿄  0.01
~~~

### 6. Token 선택

Decoding 방식에 따라 다음 Token을 선택합니다.

~~~text
서울
~~~

### 7. 다시 입력

~~~text
대한민국의 수도는 서울
~~~

을 기반으로 다시 다음 Token을 계산합니다.

이 과정을 반복하면 문장이 생성됩니다.

---

# LLM은 문장을 한 번에 만드는 것이 아니다

LLM이 긴 답변을 한 번에 완성해서 출력한다고 생각하기 쉽지만 일반적인 Autoregressive LLM은 **Token을 하나씩 생성**합니다.

~~~text
입력
 ↓
Token A 생성
 ↓
입력 + A
 ↓
Token B 생성
 ↓
입력 + A + B
 ↓
Token C 생성
 ↓
...
~~~

따라서 앞에서 생성한 내용도 다음 Token을 결정하는 문맥의 일부가 됩니다.

---

# Temperature

다음 Token을 선택할 때 항상 가장 높은 확률의 Token만 고를 필요는 없습니다.

**Temperature**는 확률 분포의 날카로움을 조절하는 데 사용됩니다.

개념적으로

~~~text
낮은 Temperature
→ 높은 확률 Token에 더 집중
→ 비교적 일관되고 보수적인 출력

높은 Temperature
→ 낮은 확률 Token도 선택될 가능성 증가
→ 더 다양하지만 불안정할 수 있는 출력
~~~

입니다.

Temperature는 Transformer 자체의 학습 구조라기보다 **생성 단계의 Decoding 설정**입니다.

---

# Top-k와 Top-p

생성 과정에서는 모든 Vocabulary Token을 그대로 후보로 사용하지 않고 후보를 제한할 수도 있습니다.

## Top-k

확률이 높은 상위 `k`개 Token만 후보로 사용합니다.

~~~text
전체 100,000 Token
       ↓
상위 50개만 선택
       ↓
그 안에서 Sampling
~~~

## Top-p

확률이 높은 순서대로 Token을 모아 누적 확률이 `p`에 도달할 때까지를 후보로 사용합니다.

예를 들어

~~~text
서울  0.50
한국  0.20
부산  0.10
도시  0.05
...
~~~

에서 `Top-p = 0.8`이라면 누적 확률 0.8 정도까지의 Token들을 후보로 사용할 수 있습니다.

이 역시 Transformer 내부 구조가 아니라 **출력 Token을 선택하는 Decoding 방법**입니다.

---

# Transformer는 어떻게 학습되는가

Transformer 역시 신경망이므로 기본적인 학습 원리는 일반 신경망과 같습니다.

~~~text
학습 Text
 ↓
Tokenization
 ↓
Transformer
 ↓
다음 Token Prediction
 ↓
정답 Token과 비교
 ↓
Loss
 ↓
Backpropagation
 ↓
Gradient
 ↓
Optimizer
 ↓
Weight Update
~~~

Attention에 사용되는

~~~text
Wq
Wk
Wv
~~~

뿐 아니라 Embedding, FFN 등의 수많은 Weight가 학습됩니다.

---

# 처음부터 Attention이 문법을 아는 것은 아니다

Transformer를 처음 만들었다고 해서 Attention Head가

~~~text
주어를 찾아라
목적어를 찾아라
앞 문장의 사람을 찾아라
~~~

같은 규칙을 알고 있는 것은 아닙니다.

처음에는 Weight가 적절하게 학습되어 있지 않습니다.

대량의 데이터를 이용해 다음 Token 예측을 반복하면서

~~~text
Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Weight Update
~~~

가 이루어지고, 그 과정에서 언어 예측에 유용한 Attention 패턴과 내부 표현이 형성됩니다.

---

# Transformer와 Parameter의 관계

LLM에서 말하는 수십억 개의 Parameter 상당수는 Transformer 내부의 행렬에 존재합니다.

예를 들어

~~~text
Embedding Matrix

Attention
 ├─ Wq
 ├─ Wk
 ├─ Wv
 └─ Output Projection

Feed Forward Network
 ├─ W1
 ├─ W2
 └─ 기타 Projection

Normalization 관련 Parameter
...
~~~

가 여러 Layer에 반복됩니다.

따라서

~~~text
7B Model
70B Model
~~~

이라는 표현은 단순히 Transformer Block의 개수가 아니라 전체 모델이 가진 학습 가능한 Parameter 규모를 나타냅니다.

---

# Parameter가 많으면 왜 성능이 좋아질 수 있는가

Parameter가 많으면 모델이 복잡한 패턴을 표현할 수 있는 용량이 증가합니다.

~~~text
작은 모델
→ 표현할 수 있는 패턴의 용량 제한

큰 모델
→ 더 복잡한 관계를 표현할 수 있는 용량 증가
~~~

하지만 Parameter만 많다고 무조건 좋은 모델이 되는 것은 아닙니다.

성능에는

~~~text
모델 구조
+
Parameter 규모
+
학습 데이터
+
데이터 품질
+
학습량
+
Optimizer
+
학습 방법
+
후처리 / 정렬
~~~

등 많은 요소가 영향을 줍니다.

---

# Context Window

Transformer가 한 번에 처리할 수 있는 Token 범위를 **Context Window**라고 합니다.

예를 들어

~~~text
8K
32K
128K
1M
~~~

등으로 표현할 수 있습니다.

Context Window 안의 Token들은 Attention을 통해 서로 관계를 계산할 수 있습니다.

~~~text
Token 1
Token 2
Token 3
...
Token N
   ↕
Attention
~~~

문맥이 길어질수록 더 많은 정보를 입력할 수 있지만 계산량과 메모리 사용량도 증가합니다.

---

# Self-Attention의 계산량 문제

기본적인 Self-Attention은 Token끼리 서로 비교합니다.

Token이 `n`개라면 Attention Score Matrix는 대략

~~~text
n × n
~~~

크기가 됩니다.

예를 들어

~~~text
1,000 Token
→ 약 1,000 × 1,000 관계

10,000 Token
→ 약 10,000 × 10,000 관계
~~~

처럼 증가합니다.

그래서 기본 Attention의 핵심 계산량은 Sequence Length에 대해 대략 **O(n²)** 특성을 가집니다.

긴 Context를 효율적으로 처리하기 위해 현대 LLM에서는 다양한 Attention 최적화와 구조가 연구되고 있습니다.

---

# KV Cache

LLM이 Token을 하나씩 생성할 때 이전 Token의 Key와 Value를 매번 다시 계산한다면 비효율적입니다.

그래서 이전 Token의 K와 V를 저장해두고 재사용합니다.

이것이 **KV Cache**입니다.

처음 Prompt를 처리할 때

~~~text
Prompt
 ↓
K1 V1
K2 V2
K3 V3
...
저장
~~~

하고 다음 Token을 생성할 때 기존 K/V를 재사용합니다.

~~~text
기존 KV Cache
      +
새 Token의 K/V
      ↓
Attention
~~~

이를 통해 Autoregressive 생성의 반복 계산을 크게 줄일 수 있습니다.

---

# Prefill과 Decode

LLM 추론은 크게 두 단계로 이해할 수 있습니다.

## Prefill

사용자가 입력한 Prompt 전체를 처리합니다.

~~~text
긴 Prompt
 ↓
Transformer
 ↓
KV Cache 생성
~~~

Prompt가 길수록 Prefill 계산량이 커집니다.

## Decode

이후 Token을 하나씩 생성합니다.

~~~text
KV Cache
+
새 Token
 ↓
다음 Token
 ↓
KV Cache 추가
 ↓
반복
~~~

그래서 LLM 서비스의 성능을 이야기할 때

~~~text
TTFT
Tokens/sec
KV Cache
Context Length
~~~

같은 개념이 중요해집니다.

---

# Transformer가 기억하는 것과 Context는 다르다

LLM에서 두 가지를 구분해야 합니다.

### Weight에 학습된 정보

~~~text
Training
 ↓
Backpropagation
 ↓
Weight 변화
~~~

학습 과정에서 Parameter에 반영된 정보입니다.

### Context에 들어 있는 정보

~~~text
현재 Prompt
대화 기록
RAG 검색 결과
System Prompt
~~~

현재 추론 과정에서 Transformer가 참고하는 정보입니다.

즉

~~~text
Weight
→ 학습을 통해 형성된 장기적인 모델 능력과 패턴

Context
→ 현재 요청에서 일시적으로 참고하는 정보
~~~

입니다.

이 차이를 이해하면 Fine-tuning과 RAG의 차이도 자연스럽게 이해할 수 있습니다.

---

# RAG와 Transformer의 관계

RAG는 Transformer 자체를 변경하는 기술이 아닙니다.

외부에서 필요한 정보를 검색해 **Context에 추가**하는 방식입니다.

~~~text
질문
 ↓
검색
 ↓
관련 문서
 ↓
Prompt에 추가
 ↓
Transformer
 ↓
답변
~~~

즉 Transformer의 Weight를 다시 학습하지 않아도 외부 정보를 참고할 수 있습니다.

---

# Fine-tuning과 Transformer의 관계

Fine-tuning은 반대로 모델의 Parameter를 추가 학습합니다.

~~~text
전문 데이터
 ↓
Transformer
 ↓
Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Weight Update
~~~

따라서

~~~text
RAG
→ Context를 바꿈

Fine-tuning
→ Parameter를 바꿈
~~~

이라는 차이가 있습니다.

---

# Transformer와 LLM은 같은 말인가

같은 말은 아닙니다.

**Transformer**는 신경망의 구조입니다.

**LLM(Large Language Model)**은 대규모 언어 데이터를 학습한 언어 모델입니다.

오늘날 많은 LLM이 Transformer를 기반으로 만들어지기 때문에 두 개념이 자주 함께 등장합니다.

~~~text
Transformer
→ 신경망 Architecture

LLM
→ 대규모 언어 모델

GPT
→ Decoder-only Transformer 기반 LLM 계열
~~~

라고 구분하면 됩니다.

---

# Transformer가 중요한 이유

Transformer가 중요한 이유는 단순히 현재 유명한 AI가 이 구조를 사용하기 때문만은 아닙니다.

Transformer를 이해하면 지금까지 따로 보이던 AI 개념이 연결됩니다.

~~~text
Token
 ↓
Embedding
 ↓
Transformer
 ├─ Attention
 ├─ Parameter
 └─ FFN
 ↓
Logits
 ↓
Next Token
~~~

학습에서는

~~~text
Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Transformer Parameter 수정
~~~

이 일어나고,

추론에서는

~~~text
Prompt
 ↓
Transformer
 ↓
Next Token
 ↓
KV Cache
 ↓
다음 Token
~~~

이 반복됩니다.

그리고 외부 지식을 넣고 싶다면

~~~text
RAG
 ↓
Context 추가
 ↓
Transformer
~~~

특정 행동이나 스타일을 학습시키고 싶다면

~~~text
Fine-tuning
 ↓
Parameter 수정
 ↓
Transformer
~~~

으로 연결됩니다.

---

# 전체 구조 한 번에 보기

현대 LLM의 전체 흐름을 단순화하면 다음과 같습니다.

~~~text
                 사용자 Text
                     ↓
                Tokenization
                     ↓
                  Token ID
                     ↓
                  Embedding
                     ↓
               Position 정보
                     ↓
          ┌─────────────────────┐
          │ Transformer Block   │
          │                     │
          │ Self-Attention      │
          │   Q = XWq           │
          │   K = XWk           │
          │   V = XWv           │
          │       ↓             │
          │ Attention           │
          │       ↓             │
          │ Feed Forward        │
          │       ↓             │
          │ Residual / Norm     │
          └─────────────────────┘
                     ↓
                여러 Layer 반복
                     ↓
                  Hidden State
                     ↓
                    Logits
                     ↓
                   Softmax
                     ↓
              Token Probability
                     ↓
                 Token 선택
                     ↓
              생성 문장에 추가
                     ↓
                 다시 반복
~~~

학습할 때는 여기에 다음 과정이 추가됩니다.

~~~text
Prediction
     ↓
정답 Token과 비교
     ↓
Loss
     ↓
Backpropagation
     ↓
Gradient
     ↓
Optimizer
     ↓
Transformer Weight 수정
~~~

---

# 핵심 개념 정리

| 개념 | 의미 |
|---|---|
| Token | Transformer가 처리하는 텍스트 단위 |
| Token ID | Token을 나타내는 숫자 |
| Embedding | Token을 벡터로 변환한 표현 |
| Position Information | Token의 순서와 위치를 표현 |
| Attention | 다른 Token을 얼마나 참고할지 계산 |
| Self-Attention | 같은 Sequence 안의 Token 관계 계산 |
| Query | 현재 필요한 정보를 표현 |
| Key | Query와 비교되는 정보 |
| Value | 실제로 전달되는 정보 |
| Multi-Head Attention | 여러 Attention을 병렬로 계산 |
| FFN | 각 Token 표현을 비선형적으로 변환 |
| Residual Connection | 입력을 출력에 더해 깊은 학습을 도움 |
| Normalization | 학습 안정성을 높이는 정규화 |
| Encoder | 입력을 문맥적인 표현으로 변환 |
| Decoder | 이전 Token을 바탕으로 다음 Token 생성 |
| Causal Mask | 미래 Token을 보지 못하도록 제한 |
| Logit | 각 Token 후보의 확률 변환 전 점수 |
| Softmax | Logit을 확률 분포로 변환 |
| Context Window | 한 번에 참고할 수 있는 Token 범위 |
| KV Cache | 이전 Token의 Key/Value를 저장해 추론 시 재사용 |
| Prefill | Prompt 전체를 처음 처리하는 단계 |
| Decode | 다음 Token을 하나씩 생성하는 단계 |

---

# 정리

Transformer의 핵심은 **Attention을 이용해 Token 사이의 관계를 계산하는 것**입니다.

입력 문장은 Token으로 나뉘고 Embedding Vector로 변환됩니다.

각 Token 표현에서

~~~text
Query
Key
Value
~~~

를 만들고

~~~text
QKᵀ
~~~

를 이용해 Token 사이의 관련도를 계산합니다.

이를 Scale하고 Softmax를 적용한 뒤 Value를 가중합합니다.

~~~text
                 QKᵀ
Attention = softmax(────)V
                  √dk
~~~

여러 Attention Head와 Feed Forward Network를 포함하는 Transformer Block을 반복해서 쌓으면 문맥을 반영한 복잡한 표현을 만들 수 있습니다.

GPT와 같은 LLM은 이러한 Transformer를 이용해

~~~text
현재 Context
 ↓
다음 Token 확률 계산
 ↓
Token 선택
 ↓
Context에 추가
 ↓
다음 Token 계산
~~~

을 반복합니다.

그리고 Transformer가 처음부터 언어를 이해하고 있는 것이 아니라 대규모 데이터에서

~~~text
Next Token Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
Weight Update
~~~

를 반복하면서 Attention, Embedding, FFN 등의 Parameter가 학습됩니다.

결국 현대 LLM의 핵심 흐름은 다음 한 줄로 정리할 수 있습니다.

> **텍스트를 Token과 Vector로 변환하고, Transformer의 Attention으로 문맥 관계를 계산한 뒤, 가장 적절한 다음 Token을 반복적으로 예측하는 신경망이다.**

Transformer를 이해하면 **Token, Embedding, Parameter, Context Window, KV Cache, Fine-tuning, RAG, LLM 학습과 추론**이 서로 어떻게 연결되는지도 함께 이해할 수 있습니다.

## 참고 자료

- Ashish Vaswani 외, [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- Jacob Devlin 외, [BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding](https://arxiv.org/abs/1810.04805)
- Alec Radford 외, [Improving Language Understanding by Generative Pre-Training](https://cdn.openai.com/research-covers/language-unsupervised/language_understanding_paper.pdf)
- Tom B. Brown 외, [Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165)
- Jianlin Su 외, [RoFormer: Enhanced Transformer with Rotary Position Embedding](https://arxiv.org/abs/2104.09864)
