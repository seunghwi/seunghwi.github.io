---
layout: single
title: "신경망 학습 원리 : 뉴런부터 가중치·손실함수·역전파까지"
categories:
  - "AI"
permalink: /topics/ai/neural-network-learning/
tags:
  - "AI"
  - "신경망"
  - "딥러닝"
  - "역전파"
  - "경사하강법"
  - "Backpropagation"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

AI가 **학습한다**는 표현을 자주 사용합니다.

사람의 학습을 생각하면 새로운 사실을 이해하고 기억하는 모습을 떠올리기 쉽지만, 인공신경망의 학습은 조금 다릅니다.

신경망에서 학습의 핵심은 의외로 단순합니다.

> **정답에 더 가까운 결과가 나오도록 신경망 내부의 숫자인 가중치(Weight)를 조금씩 수정하는 과정**

이라고 볼 수 있습니다.

예를 들어 AI에게 사진을 보여주고 고양이인지 개인지 맞히게 한다고 생각해보겠습니다.

```text
고양이 사진
   ↓
신경망
   ↓
고양이 30%
개     70%
```

실제 정답은 고양이인데 AI는 개라고 판단했습니다.

그러면 얼마나 틀렸는지를 계산하고, 그 오류가 줄어드는 방향으로 신경망 내부의 가중치를 수정합니다.

```text
예측
 ↓
정답과 비교
 ↓
오차 계산
 ↓
어떤 가중치가 오차에 영향을 줬는지 계산
 ↓
가중치 수정
 ↓
다시 예측
```

이 과정을 수없이 반복하는 것이 신경망 학습의 기본 원리입니다.

ChatGPT와 같은 LLM도 규모와 구조는 훨씬 복잡하지만 근본적인 학습 원리는 같습니다.

이 글에서는 **뉴런 → 가중치 → 순전파 → 손실함수 → 경사하강법 → 역전파** 순서로 신경망이 실제로 어떻게 학습하는지 설명합니다.

---

# 먼저 전체 구조부터 이해하기

신경망 학습의 전체 과정을 단순화하면 다음과 같습니다.

```text
학습 데이터
   ↓
신경망에 입력
   ↓
Forward Propagation
   ↓
Prediction
   ↓
정답과 비교
   ↓
Loss 계산
   ↓
Backpropagation
   ↓
Gradient 계산
   ↓
Optimizer
   ↓
Weight 수정
   ↓
다시 반복
```

결국 학습 과정에서 가장 중요한 것은 다음 세 가지입니다.

```text
현재 예측이 얼마나 틀렸는가?
          ↓
어떤 가중치가 얼마나 영향을 주었는가?
          ↓
가중치를 어느 방향으로 얼마나 수정할 것인가?
```

이 세 가지 질문을 해결하기 위해 각각 다음 개념이 사용됩니다.

| 질문                | 관련 개념                       |
| ----------------- | --------------------------- |
| 얼마나 틀렸는가          | Loss Function               |
| 어떤 값이 얼마나 영향을 줬는가 | Gradient, Backpropagation   |
| 어떻게 수정할 것인가       | Optimizer, Gradient Descent |

이 관계를 먼저 기억해두면 이후 내용이 훨씬 쉽게 연결됩니다.

---

# 인공신경망이란 무엇인가

**인공신경망(Artificial Neural Network)**은 여러 개의 계산 단위를 연결해 입력을 원하는 출력으로 변환하는 모델입니다.

사람의 신경세포에서 아이디어를 얻었기 때문에 각 계산 단위를 **Neuron 또는 Node**라고 부릅니다.

가장 단순한 형태는 다음과 같습니다.

```text
입력
 ↓
뉴런
 ↓
출력
```

하지만 실제 신경망은 많은 뉴런을 층으로 구성합니다.

```text
Input Layer
     ↓
Hidden Layer
     ↓
Hidden Layer
     ↓
Output Layer
```

예를 들어 입력이 세 개라면 다음과 같은 구조를 생각할 수 있습니다.

```text
x1 ─┐
    ├─→ Neuron ─→ Output
x2 ─┤
    │
x3 ─┘
```

뉴런은 입력을 그대로 전달하는 것이 아니라 각각의 입력에 **가중치(Weight)**를 적용합니다.

---

# 가중치 Weight란 무엇인가

신경망에서 가장 중요한 개념 중 하나가 **가중치(Weight)**입니다.

입력이

```text
x1
x2
x3
```

이고 각각의 가중치가

```text
w1
w2
w3
```

라면 뉴런에서는 다음과 같은 계산을 합니다.

```text
x1 × w1
+
x2 × w2
+
x3 × w3
```

수식으로 표현하면

```text
z = x1w1 + x2w2 + x3w3
```

입니다.

가중치는 각 입력이 결과에 **얼마나 강하게 영향을 줄 것인지**를 결정합니다.

예를 들어 집값을 예측하는 AI가 있다고 생각해보겠습니다.

입력 데이터가 다음과 같다고 해보겠습니다.

```text
집 크기
역과의 거리
건축 연도
```

학습 결과 모델이 `집 크기`를 매우 중요하게 판단한다면 해당 입력과 연결된 가중치가 결과에 큰 영향을 미치는 방향으로 조정될 수 있습니다.

즉 신경망의 학습은 결국 수많은 가중치를 적절한 값으로 찾는 과정이라고 볼 수 있습니다.

---

# Parameter란 무엇인가

AI에서 자주 듣는 **Parameter**가 바로 이런 학습 가능한 숫자입니다.

대표적으로

* Weight
* Bias

등이 Parameter에 해당합니다.

예를 들어 어떤 모델이

```text
7B Parameters
```

라고 한다면 약 **70억 개의 학습 가능한 파라미터**를 가지고 있다는 의미입니다.

LLM에서는 Transformer 내부의 여러 행렬이 이런 파라미터로 구성됩니다.

Transformer에서 보았던

```text
Wq
Wk
Wv
```

역시 학습되는 Weight Matrix입니다.

따라서

````text
AI가 학습한다
```

는 말을 좀 더 기술적으로 표현하면

~~~text
모델의 Parameter를 조정한다
````

라고 할 수 있습니다.

---

# Bias란 무엇인가

뉴런에서는 Weight뿐 아니라 **Bias**라는 값도 사용합니다.

계산은 다음과 같습니다.

```text
z = x1w1 + x2w2 + x3w3 + b
```

여기서

```text
b = Bias
```

입니다.

Bias는 입력과 관계없이 결과를 일정량 이동시키는 값입니다.

아주 단순하게

```text
y = wx
```

라는 식이 있다면 반드시 원점을 통과합니다.

반면

```text
y = wx + b
```

는 Bias인 `b`를 이용해 그래프를 위아래로 이동시킬 수 있습니다.

즉 Weight가 입력의 **영향력과 기울기**를 조절한다면 Bias는 판단의 **기준 위치를 이동시키는 역할**을 한다고 이해할 수 있습니다.

Bias 역시 학습 과정에서 조정되는 Parameter입니다.

---

# 뉴런 하나의 실제 계산

뉴런 하나를 단순화하면 다음과 같습니다.

```text
x1 ──× w1──┐
           │
x2 ──× w2──┼─→ 합산 → + Bias → Activation → Output
           │
x3 ──× w3──┘
```

수식으로 표현하면

```text
z = Σ(xiwi) + b

y = activation(z)
```

입니다.

여기서 새로운 개념인 **Activation Function**이 등장합니다.

---

# Activation Function이 필요한 이유

여러 개의 선형 계산만 계속 연결하면 신경망을 아무리 깊게 만들어도 결국 하나의 선형 계산으로 표현할 수 있습니다.

예를 들어

```text
y = ax + b
```

형태의 계산을 여러 층 연결해도 전체적으로는 다시 선형적인 관계가 됩니다.

하지만 현실의 문제는 대부분 단순한 직선 관계가 아닙니다.

이미지, 음성, 자연어처럼 복잡한 패턴을 학습하려면 **비선형성(Non-linearity)**이 필요합니다.

이를 추가하는 것이 Activation Function입니다.

대표적인 활성화 함수에는 다음과 같은 것들이 있습니다.

* Sigmoid
* Tanh
* ReLU
* GELU
* SiLU / Swish

---

# ReLU

가장 이해하기 쉬운 활성화 함수 중 하나가 **ReLU(Rectified Linear Unit)**입니다.

```text
ReLU(x) = max(0, x)
```

즉

```text
입력이 음수 → 0
입력이 양수 → 그대로
```

입니다.

예를 들어

```text
ReLU(-3) = 0
ReLU(2)  = 2
```

가 됩니다.

이처럼 단순한 비선형 함수를 신경망 사이에 넣으면 여러 층을 조합하면서 훨씬 복잡한 관계를 표현할 수 있게 됩니다.

현대 Transformer에서는 ReLU뿐 아니라 GELU, SiLU, SwiGLU 등의 방식도 사용됩니다.

---

# Layer란 무엇인가

뉴런 여러 개를 묶은 것을 **Layer**라고 합니다.

일반적인 신경망은 크게 세 종류의 Layer로 설명할 수 있습니다.

```text
Input Layer
     ↓
Hidden Layer
     ↓
Output Layer
```

### Input Layer

외부 데이터를 받는 부분입니다.

예를 들어

```text
x1 = 온도
x2 = 습도
x3 = 기압
```

같은 입력이 들어옵니다.

### Hidden Layer

입력을 변환하면서 패턴을 학습하는 부분입니다.

```text
Input
 ↓
Hidden 1
 ↓
Hidden 2
 ↓
Hidden 3
```

층이 깊어질수록 여러 단계의 복잡한 표현을 만들 수 있습니다.

### Output Layer

최종 예측 결과를 만드는 부분입니다.

예를 들어

```text
고양이 0.92
개     0.06
새     0.02
```

처럼 출력할 수 있습니다.

---

# Deep Learning의 Deep은 무엇인가

**Deep Learning**에서 Deep은 Hidden Layer를 여러 층 사용하는 깊은 신경망에서 나온 표현입니다.

```text
Input
 ↓
Layer
 ↓
Layer
 ↓
Layer
 ↓
Layer
 ↓
...
 ↓
Output
```

층을 깊게 쌓으면 각 층이 이전 층에서 만들어진 정보를 다시 변환하면서 복잡한 특징을 표현할 수 있습니다.

이미지 인식을 아주 단순화하면 초기 층에서는

```text
선
모서리
방향
```

같은 단순한 특징을 처리하고 더 깊은 층에서는

```text
눈
귀
얼굴
물체
```

처럼 복잡한 특징과 관련된 표현이 형성될 수 있습니다.

다만 실제 딥러닝 모델 내부의 표현이 사람이 정한 이런 단계로 정확히 나뉘는 것은 아닙니다.

---

# Forward Propagation

이제 실제 학습을 시작해보겠습니다.

입력 데이터를 신경망에 넣고 앞쪽 Layer에서 뒤쪽 Layer로 계산을 진행해 결과를 만드는 과정을 **Forward Propagation, 순전파**라고 합니다.

```text
Input
 ↓
Layer 1
 ↓
Layer 2
 ↓
Layer 3
 ↓
Prediction
```

예를 들어 입력이

```text
x = 2
```

이고 Weight가

```text
w = 3
```

이라면 단순한 모델에서는

```text
y = x × w

y = 2 × 3

y = 6
```

이라는 예측 결과가 나옵니다.

실제 신경망에서는 이런 계산이 수많은 뉴런과 행렬을 통해 수행됩니다.

중요한 것은 순전파 단계에서는 **현재 가지고 있는 Weight를 이용해서 일단 답을 만들어본다**는 것입니다.

---

# Prediction과 정답 비교

모델이 결과를 만들었다면 실제 정답과 비교해야 합니다.

예를 들어 정답이

```text
10
```

인데 모델이

```text
6
```

이라고 예측했다면 틀렸습니다.

```text
정답      = 10
Prediction = 6
```

그렇다면 다음 질문이 생깁니다.

> 얼마나 틀렸는가?

이를 숫자로 계산하는 것이 **Loss Function**입니다.

---

# Loss Function이란 무엇인가

**Loss Function, 손실 함수**는 모델의 예측이 정답과 얼마나 다른지를 숫자로 표현합니다.

개념적으로

```text
정답과 매우 비슷함
→ Loss 작음

정답과 많이 다름
→ Loss 큼
```

입니다.

학습의 목표는 결국

```text
Loss ↓
```

즉 **Loss를 최소화하는 Parameter를 찾는 것**입니다.

---

# 가장 단순한 Loss 예제

회귀 문제에서는 이해하기 쉬운 예로 **Mean Squared Error(MSE)**를 사용할 수 있습니다.

예측과 정답의 차이를 제곱합니다.

```text
Loss = (Prediction - Target)²
```

예를 들어

```text
Prediction = 6
Target     = 10
```

이면

```text
Loss
= (6 - 10)²
= 16
```

입니다.

Weight를 수정한 뒤 예측이 9가 됐다면

```text
Loss
= (9 - 10)²
= 1
```

이 됩니다.

```text
처음 Loss = 16

학습 후 Loss = 1
```

모델이 더 좋은 방향으로 바뀌었다고 볼 수 있습니다.

---

# Loss와 Accuracy는 다르다

Loss와 Accuracy를 같은 개념으로 생각하면 안 됩니다.

Accuracy는

```text
몇 개를 맞혔는가?
```

를 나타내는 평가 지표입니다.

Loss는

```text
모델을 어느 방향으로 얼마나 수정해야 하는가?
```

를 계산할 수 있도록 만든 연속적인 학습 신호입니다.

예를 들어 두 모델이 모두 정답을 맞혔더라도

```text
모델 A
고양이 51%
개     49%

모델 B
고양이 99%
개      1%
```

처럼 예측의 확신 정도가 다를 수 있습니다.

단순 Accuracy에서는 둘 다 정답이지만 Loss는 다르게 계산될 수 있습니다.

---

# LLM에서는 어떤 Loss를 사용하는가

LLM은 주로 **다음 Token을 예측**합니다.

예를 들어

```text
대한민국의 수도는
```

다음에

```text
서울
```

이 나와야 한다고 해보겠습니다.

모델이 다음과 같이 예측했다고 가정합니다.

```text
서울  → 0.10
부산  → 0.50
대전  → 0.20
기타  → 0.20
```

정답인 `서울`의 확률이 매우 낮습니다.

따라서 Loss가 커집니다.

학습이 진행되어

```text
서울  → 0.95
부산  → 0.02
대전  → 0.01
기타  → 0.02
```

가 된다면 Loss가 작아집니다.

언어 모델 학습에서는 대표적으로 **Cross-Entropy Loss**를 사용합니다.

핵심은 똑같습니다.

> 정답 Token의 확률이 높아지도록 Weight를 수정한다.

---

# 그런데 Weight를 어떻게 수정할까

여기서 신경망 학습의 핵심 문제가 등장합니다.

Loss가

```text
16
```

이라고 해서 Weight를 어떻게 바꿔야 할지는 아직 알 수 없습니다.

현재 Weight가

```text
3.0
```

이라면

```text
3.1로 올려야 하는가?

2.9로 내려야 하는가?

얼마나 바꿔야 하는가?
```

를 알아야 합니다.

이 문제를 해결하는 핵심 개념이 **미분과 Gradient**입니다.

---

# 미분을 왜 사용하는가

미분은 특정 값을 조금 바꿨을 때 결과가 얼마나 변하는지를 알려줍니다.

예를 들어 Loss가 Weight에 따라 다음처럼 변한다고 생각해보겠습니다.

```text
Loss
 ↑
 │\
 │ \
 │  \
 │   \      /
 │    \____/
 │
 └────────────→ Weight
```

우리가 원하는 것은 가장 낮은 지점입니다.

```text
Loss 최소
```

현재 위치에서 기울기를 알 수 있다면 어느 방향으로 이동해야 Loss가 작아지는지 알 수 있습니다.

```text
기울기 양수
→ Weight를 줄이는 방향

기울기 음수
→ Weight를 늘리는 방향
```

이 기울기가 **Gradient**입니다.

---

# Gradient란 무엇인가

Gradient는 Parameter를 조금 변화시켰을 때 Loss가 어느 방향으로 얼마나 변하는지를 나타냅니다.

Weight 하나만 있다면

```text
dLoss / dWeight
```

처럼 표현할 수 있습니다.

예를 들어

```text
Gradient = +4
```

라면 Weight가 증가할수록 Loss가 증가하는 방향이라는 의미입니다.

따라서 Loss를 줄이려면 반대 방향으로 이동해야 합니다.

```text
Weight ↓
```

반대로

```text
Gradient = -4
```

라면 Weight를 증가시키는 방향으로 이동하면 Loss를 줄일 수 있습니다.

---

# Gradient Descent

Gradient의 반대 방향으로 Parameter를 조금씩 움직여 Loss를 줄이는 방법을 **Gradient Descent, 경사하강법**이라고 합니다.

```text
높은 Loss
   ●
    \
     ●
      \
       ●
        \
         ●
          \___★ 최소점
```

산에서 아래쪽으로 조금씩 내려가는 모습과 비슷합니다.

Weight 업데이트 식은 기본적으로 다음과 같습니다.

```text
새 Weight
=
기존 Weight
-
Learning Rate × Gradient
```

또는

```text
w = w - η × ∂L/∂w
```

로 표현합니다.

여기서

```text
η = Learning Rate
```

입니다.

---

# Learning Rate

**Learning Rate, 학습률**은 한 번 학습할 때 Weight를 얼마나 크게 수정할지 결정합니다.

예를 들어

```text
Weight = 3.0
Gradient = 2.0
Learning Rate = 0.1
```

이라면

```text
새 Weight

= 3.0 - (0.1 × 2.0)

= 2.8
```

이 됩니다.

Learning Rate가 너무 크면

```text
최소점
   ↓
───●────────
  ↙ ↘
●     ●
 ↘   ↙
   ●
```

처럼 최소점을 계속 지나칠 수 있습니다.

반대로 너무 작으면

```text
●
 ↓
●
 ↓
●
 ↓
●
 ↓
...
```

학습이 지나치게 느려집니다.

따라서 Learning Rate는 딥러닝 학습에서 매우 중요한 **Hyperparameter**입니다.

---

# Parameter와 Hyperparameter의 차이

둘은 구분해야 합니다.

### Parameter

모델이 학습을 통해 직접 변경하는 값입니다.

```text
Weight
Bias
```

등이 대표적입니다.

### Hyperparameter

사람이 학습 방법을 결정하기 위해 설정하는 값입니다.

대표적으로

```text
Learning Rate
Batch Size
Epoch
Optimizer 설정
```

등이 있습니다.

즉

```text
Parameter
→ AI가 학습하면서 찾음

Hyperparameter
→ 학습 방식을 사람이 설정
```

이라고 이해하면 됩니다.

---

# Backpropagation

지금까지는 Weight가 하나인 단순한 상황을 생각했습니다.

하지만 실제 신경망에는 수백만, 수십억 개 이상의 Parameter가 있을 수 있습니다.

```text
Input
 ↓
Layer 1
 ↓
Layer 2
 ↓
Layer 3
 ↓
Output
 ↓
Loss
```

Loss가 발생했을 때 각 Parameter가 Loss에 얼마나 영향을 줬는지를 모두 계산해야 합니다.

이를 효율적으로 계산하는 방법이 **Backpropagation, 역전파**입니다.

---

# 왜 이름이 역전파인가

예측할 때는 입력에서 출력 방향으로 계산합니다.

```text
Input
  ↓
Layer 1
  ↓
Layer 2
  ↓
Layer 3
  ↓
Output
```

이것이 **Forward Propagation**입니다.

반면 Loss가 계산된 후 Gradient는 계산 그래프를 따라 뒤에서 앞으로 전달됩니다.

```text
Loss
 ↑
Output
 ↑
Layer 3
 ↑
Layer 2
 ↑
Layer 1
 ↑
Input 방향
```

그래서 **Back Propagation**, 즉 역전파라고 부릅니다.

---

# 역전파의 핵심은 Chain Rule

역전파에서 중요한 수학적 원리는 미분의 **Chain Rule, 연쇄법칙**입니다.

예를 들어

```text
x
 ↓
a
 ↓
b
 ↓
Loss
```

라는 계산이 있다고 해보겠습니다.

x가 Loss에 얼마나 영향을 주는지 알고 싶다면 중간 관계를 연결해서 계산할 수 있습니다.

```text
dLoss/dx

=

dLoss/db
×
db/da
×
da/dx
```

즉

```text
x가 a에 얼마나 영향을 주는가

×

a가 b에 얼마나 영향을 주는가

×

b가 Loss에 얼마나 영향을 주는가
```

를 연결합니다.

신경망도 같은 원리를 사용합니다.

```text
Weight
 ↓
Neuron
 ↓
Layer
 ↓
Layer
 ↓
Prediction
 ↓
Loss
```

역전파를 통해

```text
각 Weight가 최종 Loss에 얼마나 영향을 줬는가
```

를 계산할 수 있습니다.

---

# 아주 단순한 학습 예제

다음 모델이 있다고 해보겠습니다.

```text
y = wx
```

학습 데이터는

```text
x = 2
정답 y = 10
```

이고 초기 Weight는

```text
w = 3
```

이라고 하겠습니다.

### 1. Forward

```text
Prediction
= 2 × 3
= 6
```

### 2. Loss

```text
Loss
= (6 - 10)²
= 16
```

### 3. Gradient 계산

Loss는

```text
L = (wx - y)²
```

이므로 Weight에 대한 Gradient는

```text
dL/dw
=
2(wx-y)x
```

입니다.

값을 넣으면

```text
2 × (6-10) × 2

= -16
```

입니다.

### 4. Weight 수정

Learning Rate를 단순히

```text
0.01
```

이라고 하면

```text
새 Weight

= 3 - (0.01 × -16)

= 3.16
```

이 됩니다.

### 5. 다시 예측

```text
Prediction

= 2 × 3.16

= 6.32
```

정답 10에 조금 가까워졌습니다.

이 과정을 반복합니다.

```text
6
 ↓
6.32
 ↓
6.61
 ↓
...
 ↓
10에 가까워짐
```

이것이 신경망 학습의 가장 기본적인 모습입니다.

---

# Optimizer란 무엇인가

실제 딥러닝에서는 단순 Gradient Descent보다 효율적으로 Parameter를 업데이트하기 위해 **Optimizer**를 사용합니다.

대표적인 Optimizer에는 다음이 있습니다.

* SGD
* Momentum
* RMSProp
* Adam
* AdamW

기본 아이디어는 같습니다.

```text
Gradient 계산
      ↓
Optimizer
      ↓
Weight 업데이트
```

예를 들어 Adam은 과거 Gradient의 이동 평균 등을 이용해 Parameter마다 업데이트 크기를 조정합니다.

Transformer와 LLM 학습에서는 **AdamW** 계열 Optimizer가 널리 사용됩니다.

Optimizer는 새로운 정답을 만들어주는 것이 아니라 **Gradient 정보를 이용해 Weight를 어떻게 업데이트할지 결정하는 알고리즘**입니다.

---

# Batch란 무엇인가

학습 데이터가 10억 개 있다고 해서 모든 데이터를 한꺼번에 GPU에 넣을 수는 없습니다.

따라서 데이터를 작은 묶음으로 나눠 학습합니다.

이를 **Batch**라고 합니다.

예를 들어 데이터가 1,000개이고

```text
Batch Size = 100
```

이라면

```text
Batch 1 → 데이터 1~100
Batch 2 → 데이터 101~200
Batch 3 → 데이터 201~300
...
Batch 10 → 데이터 901~1000
```

처럼 처리할 수 있습니다.

각 Batch에서 대략

```text
Forward
 ↓
Loss
 ↓
Backward
 ↓
Weight Update
```

과정이 수행됩니다.

---

# Mini-Batch를 사용하는 이유

데이터 하나씩 Weight를 업데이트하면 계산 효율이 떨어지고 Gradient 변동이 커질 수 있습니다.

반대로 전체 데이터를 한 번에 사용하면 메모리가 많이 필요합니다.

그래서 일반적으로 여러 데이터를 묶은 **Mini-Batch**를 사용합니다.

```text
데이터 일부
   ↓
GPU 병렬 계산
   ↓
평균 Loss
   ↓
Gradient
   ↓
Weight Update
```

GPU가 AI 학습에 매우 적합한 이유 중 하나도 많은 행렬 연산과 Batch 데이터를 병렬로 처리할 수 있기 때문입니다.

---

# Epoch란 무엇인가

전체 학습 데이터를 한 번 모두 사용하면 **1 Epoch**라고 합니다.

예를 들어 데이터가 10,000개라면

```text
10,000개 전체 학습 완료
=
1 Epoch
```

입니다.

같은 데이터를 다시 학습하면

```text
2 Epoch
```

가 됩니다.

```text
전체 데이터
 ↓
Epoch 1
 ↓
전체 데이터 다시
 ↓
Epoch 2
 ↓
전체 데이터 다시
 ↓
Epoch 3
```

Epoch를 늘리면 모델이 데이터를 더 많이 학습하지만 무조건 좋은 것은 아닙니다.

너무 많이 학습하면 **Overfitting**이 발생할 수 있습니다.

---

# Step이란 무엇인가

Batch 하나를 이용해 한 번 Parameter를 업데이트하는 단위를 보통 **Step**이라고 합니다.

예를 들어

```text
전체 데이터 = 10,000개
Batch Size = 100
```

이라면 대략

```text
1 Epoch = 100 Steps
```

가 됩니다.

따라서

```text
Epoch
→ 전체 데이터를 몇 번 보았는가

Batch Size
→ 한 번에 몇 개를 처리하는가

Step
→ Weight를 몇 번 업데이트했는가
```

라고 구분하면 이해하기 쉽습니다.

LLM처럼 데이터가 매우 큰 학습에서는 Epoch보다 **학습 Step이나 처리한 Token 수**를 기준으로 학습 규모를 표현하기도 합니다.

---

# Training / Validation / Test

AI 모델을 만들 때 데이터를 일반적으로 세 종류로 나눕니다.

```text
전체 데이터
   │
   ├─ Training Set
   ├─ Validation Set
   └─ Test Set
```

### Training Set

실제로 Weight를 학습하는 데이터입니다.

### Validation Set

학습 중 모델이 처음 보는 데이터에서도 잘 작동하는지 확인하고 Hyperparameter나 모델 선택에 활용합니다.

### Test Set

최종 모델의 일반화 성능을 평가하기 위해 사용합니다.

Test 데이터를 학습 과정에서 계속 보면서 모델을 수정하면 Test가 더 이상 공정한 최종 평가 데이터가 아니게 됩니다.

---

# Overfitting

모델이 Training Data에 지나치게 맞춰지는 것을 **Overfitting, 과적합**이라고 합니다.

예를 들어

```text
Training Accuracy = 99%

Validation Accuracy = 70%
```

라면 학습 데이터는 매우 잘 맞히지만 새로운 데이터에서는 성능이 크게 떨어지고 있을 수 있습니다.

쉽게 말하면

> 문제의 원리를 배운 것이 아니라 학습 문제를 지나치게 외운 상태

와 비슷합니다.

```text
Training Loss
계속 감소 ↓↓↓

Validation Loss
감소하다 다시 증가 ↑
```

와 같은 패턴이 나타날 수 있습니다.

---

# Underfitting

반대로 모델이 데이터의 패턴을 충분히 학습하지 못한 상태를 **Underfitting, 과소적합**이라고 합니다.

```text
Training 성능도 낮음
Validation 성능도 낮음
```

이라면 모델의 표현력이 부족하거나 학습이 충분하지 않은 상황 등을 의심할 수 있습니다.

```text
Underfitting
     ↓
학습 자체가 부족

Good Fit
     ↓
새 데이터에도 잘 일반화

Overfitting
     ↓
학습 데이터를 지나치게 외움
```

---

# Overfitting을 줄이는 방법

대표적으로 다음과 같은 방법을 사용할 수 있습니다.

* 더 다양하고 충분한 데이터
* Data Augmentation
* Weight Decay
* Dropout
* Early Stopping
* 모델 크기 조절
* 적절한 학습 횟수
* Validation 성능 관찰

LLM에서는 데이터 중복 제거와 데이터 품질 관리도 매우 중요합니다.

---

# Weight Decay

Weight가 지나치게 큰 값으로 커지는 것을 억제하는 Regularization 방법 중 하나입니다.

Optimizer가 Parameter를 업데이트할 때 Weight를 조금 줄이는 방향의 효과를 추가합니다.

현대 Transformer 학습에서 많이 사용되는 **AdamW**의 `W`도 Weight Decay와 관련되어 있습니다.

---

# Dropout

학습할 때 일부 뉴런 또는 연결의 출력을 확률적으로 사용하지 않는 방법입니다.

```text
● → 사용
● → 사용 안 함
● → 사용
● → 사용
● → 사용 안 함
```

특정 뉴런에 지나치게 의존하는 것을 줄여 일반화 성능을 높이는 데 도움을 줄 수 있습니다.

일반적으로 추론 시에는 학습 때와 같은 방식으로 Dropout을 적용하지 않습니다.

---

# 학습과 추론의 차이

이제 매우 중요한 차이를 이해할 수 있습니다.

## Training

학습에서는

```text
Forward
 ↓
Loss
 ↓
Backward
 ↓
Weight Update
```

가 모두 필요합니다.

즉 Parameter를 계속 수정합니다.

## Inference

이미 학습된 모델을 사용할 때는 일반적으로

```text
Input
 ↓
Forward
 ↓
Output
```

만 수행합니다.

Weight를 수정하지 않습니다.

따라서

```text
Training
→ 답을 만들고 틀린 정도를 계산해 Weight를 수정

Inference
→ 학습된 Weight를 사용해서 답만 계산
```

입니다.

이것이 AI에서 말하는 **학습과 추론의 가장 중요한 차이**입니다.

---

# 왜 AI 학습에는 GPU가 필요한가

신경망의 대부분 계산은 행렬 연산입니다.

예를 들어 한 Layer는 단순화하면

```text
Y = XW + b
```

와 같이 표현할 수 있습니다.

Transformer에서도

```text
Q = XWq
K = XWk
V = XWv
```

처럼 대규모 행렬 연산을 수행합니다.

GPU는 많은 단순한 연산을 동시에 처리하는 데 매우 강합니다.

```text
CPU

연산
연산
연산
연산


GPU

연산 연산 연산 연산
연산 연산 연산 연산
연산 연산 연산 연산
연산 연산 연산 연산
```

실제 구조는 이보다 훨씬 복잡하지만 AI 학습 관점에서는 **대량의 행렬 계산을 병렬로 처리할 수 있다는 점**이 핵심입니다.

또한 학습에서는 Weight뿐 아니라 다음과 같은 값들을 메모리에 유지해야 합니다.

```text
Weight
Gradient
Optimizer State
Activation
```

그래서 같은 모델이라도 일반적으로 추론보다 학습에 훨씬 많은 GPU 메모리가 필요합니다.

---

# LLM도 결국 같은 방식으로 학습한다

ChatGPT와 같은 LLM을 보면 매우 복잡해 보이지만 지금까지 설명한 원리는 그대로 적용됩니다.

차이는 규모와 신경망 구조입니다.

일반적인 신경망에서는

```text
Input
 ↓
Neural Network
 ↓
Prediction
 ↓
Loss
```

였다면 LLM에서는

```text
Text
 ↓
Tokenization
 ↓
Embedding
 ↓
Transformer
 ↓
다음 Token 확률
 ↓
정답 Token과 비교
 ↓
Loss
 ↓
Backpropagation
 ↓
Transformer Weight 수정
```

이 됩니다.

---

# Transformer의 Weight도 이렇게 학습된다

Transformer에서 Self-Attention을 계산할 때

```text
Q = XWq
K = XWk
V = XWv
```

를 사용했습니다.

처음부터

```text
Wq
Wk
Wv
```

에 완벽한 값이 들어 있는 것이 아닙니다.

학습을 통해 수정됩니다.

```text
문장 입력
 ↓
Transformer
 ↓
다음 Token 예측
 ↓
정답 Token과 비교
 ↓
Loss
 ↓
Backpropagation
 ↓
Wq, Wk, Wv 및 다른 Parameter의 Gradient 계산
 ↓
Optimizer
 ↓
Parameter 수정
```

이 과정을 막대한 양의 텍스트에 대해 반복합니다.

그 결과 Attention을 비롯한 Transformer 내부 계산이 다음 Token을 더 잘 예측할 수 있는 방향으로 변화합니다.

---

# AI에게 사람이 문법 규칙을 직접 넣는 것은 아니다

여기서 현대 AI의 중요한 특징을 이해할 수 있습니다.

사람이 Transformer에게 일일이

```text
'나는'은 주어다.

'먹었다'는 동사다.

주어와 동사를 연결해라.

'그'가 나오면 앞의 사람을 찾아라.
```

같은 규칙을 모두 프로그래밍하지 않습니다.

대신 많은 학습 예제를 제공합니다.

```text
입력
 ↓
예측
 ↓
틀림
 ↓
Loss
 ↓
Weight 수정
 ↓
다시 예측
```

이 과정이 반복되면서 언어를 예측하는 데 필요한 패턴이 Parameter에 반영됩니다.

따라서 딥러닝은 전통적인 프로그램과 근본적으로 다른 특징을 가집니다.

전통적인 프로그램은

```text
사람이 규칙 작성
       ↓
프로그램
       ↓
결과
```

라면 머신러닝은

```text
데이터 + 학습 목표
       ↓
학습
       ↓
Parameter
       ↓
모델
```

에 가깝습니다.

---

# 그렇다면 AI의 지식은 어디에 있는가

이 질문도 이제 어느 정도 이해할 수 있습니다.

LLM이 학습한 내용을 일반 데이터베이스처럼

```text
서울 = 대한민국 수도
사과 = 과일
고양이 = 동물
```

형태로 저장하는 것은 아닙니다.

학습 과정에서 수많은 Parameter가 조정되면서 여러 개념과 패턴이 **분산된 형태로 모델 내부에 반영**됩니다.

```text
학습 데이터
     ↓
Loss
     ↓
Backpropagation
     ↓
수많은 Parameter 수정
     ↓
모델 내부 표현과 행동 변화
```

그래서 특정 사실 하나가 Parameter 하나에 저장되어 있다고 생각하면 안 됩니다.

하나의 Parameter가 여러 계산에 영향을 줄 수 있고 하나의 개념 역시 수많은 Parameter의 상호작용으로 표현될 수 있습니다.

이 때문에 LLM의 내부 지식은 일반적인 데이터베이스처럼 특정 주소에서 정확히 꺼내는 구조와 다릅니다.

---

# 그렇다면 Fine-tuning은 무엇인가

Fine-tuning 역시 기본 원리는 같습니다.

이미 학습된 모델이 있다고 해보겠습니다.

```text
Pre-trained Model
```

여기에 특정 목적의 데이터를 추가로 제공합니다.

```text
기본 모델
   +
전문 데이터
   ↓
Forward
   ↓
Loss
   ↓
Backpropagation
   ↓
Weight 수정
```

즉 **이미 학습된 Parameter를 특정 목적에 맞게 추가로 조정하는 것**이 Fine-tuning입니다.

처음부터 모든 것을 학습하는 것이 아니라 이미 많은 것을 배운 모델에서 시작한다는 차이가 있습니다.

---

# LoRA도 결국 Weight 학습이다

LoRA도 기본적인 학습 원리는 다르지 않습니다.

Full Fine-tuning에서는 많은 기존 Weight를 직접 수정하지만 LoRA에서는 원래 Weight를 고정하고 작은 추가 행렬을 학습합니다.

```text
Original Weight
      ↓
     고정

LoRA A
LoRA B
      ↓
     학습
```

하지만 LoRA의 A와 B 역시

```text
Forward
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
```

라는 동일한 원리로 학습됩니다.

따라서 **신경망의 기본 학습 원리를 이해하면 Pre-training, Fine-tuning, LoRA의 공통 원리도 함께 이해할 수 있습니다.**

---

# Pre-training은 무엇이 다른가

LLM의 Pre-training에서는 엄청난 양의 텍스트를 사용해 기본적인 언어 모델 능력을 학습합니다.

예를 들어

```text
대한민국의 수도는 서울이다
```

라는 문장이 있다면 학습용 입력과 정답을 다음처럼 만들 수 있습니다.

```text
대한민국의
→ 수도는

대한민국의 수도는
→ 서울

대한민국의 수도는 서울
→ 이다
```

실제 학습은 Token 단위로 이루어지며 한 Sequence 안의 여러 위치에 대한 다음 Token Loss를 효율적으로 함께 계산할 수 있습니다.

전체적으로 보면

```text
대규모 Text
   ↓
Tokenization
   ↓
Transformer
   ↓
Next Token Prediction
   ↓
Loss
   ↓
Backpropagation
   ↓
Weight Update
   ↓
수많은 데이터에서 반복
```

입니다.

이 과정에서 모델은 언어의 통계적 패턴뿐 아니라 데이터에 포함된 다양한 개념과 관계를 다음 Token 예측에 유용한 형태로 학습하게 됩니다.

---

# 학습했다고 항상 정확한 것은 아니다

신경망의 학습 목표를 이해하면 LLM이 틀린 답을 만들 수 있는 이유도 이해하기 쉬워집니다.

LLM의 기본 학습 목표는 대체로

> **주어진 문맥에서 다음 Token의 확률을 잘 예측하는 것**

입니다.

모델 내부에 모든 사실을 데이터베이스처럼 정확하게 저장하고 검색하는 것이 아닙니다.

따라서

```text
언어적으로 자연스러운 답
≠
항상 사실인 답
```

입니다.

이것이 LLM에서 Hallucination 문제가 발생하는 근본적인 이유 중 하나입니다.

최신 정보나 정확한 사내 문서를 사용해야 한다면 RAG나 외부 도구를 연결하는 이유도 여기에 있습니다.

---

# 학습 과정을 한 번에 정리하면

신경망 학습의 전체 과정은 다음과 같습니다.

```text
              학습 데이터
                  ↓
                Input
                  ↓
          Forward Propagation
                  ↓
              Prediction
                  ↓
          정답과 Prediction 비교
                  ↓
             Loss Function
                  ↓
            Backpropagation
                  ↓
               Gradient
                  ↓
              Optimizer
                  ↓
            Weight Update
                  ↓
                  └──────────┐
                             │
                  다시 Forward
```

이 과정을 반복하면서

```text
Loss ↓
Prediction 정확도 ↑
```

가 되도록 Parameter를 찾아갑니다.

---

# 핵심 개념의 관계

| 개념                  | 역할                          |
| ------------------- | --------------------------- |
| Neuron              | 입력을 받아 계산하는 기본 단위           |
| Weight              | 입력이 결과에 미치는 영향 조절           |
| Bias                | 계산의 기준점을 조절                 |
| Parameter           | 학습으로 변경되는 값                 |
| Activation Function | 비선형성을 추가                    |
| Layer               | 여러 계산 단위를 묶은 층              |
| Forward Propagation | 현재 Weight로 결과 계산            |
| Prediction          | 모델이 만든 결과                   |
| Loss                | 예측이 얼마나 틀렸는지 표현             |
| Gradient            | Parameter 변화가 Loss에 미치는 영향  |
| Backpropagation     | Gradient를 뒤에서부터 효율적으로 계산    |
| Optimizer           | Gradient를 이용해 Parameter 수정  |
| Learning Rate       | 한 번에 Parameter를 얼마나 수정할지 결정 |
| Batch               | 한 번의 계산에 사용하는 데이터 묶음        |
| Step                | Parameter를 한 번 업데이트하는 단위    |
| Epoch               | 전체 학습 데이터를 한 번 사용           |
| Overfitting         | 학습 데이터에 지나치게 맞춰진 상태         |
| Inference           | 학습된 Parameter로 결과만 계산       |

---

# Transformer와 연결해서 보면

신경망의 기본 학습 구조와 Transformer는 서로 다른 원리가 아닙니다.

Transformer 역시 하나의 거대한 신경망입니다.

```text
Token
 ↓
Embedding
 ↓
Transformer Block
 │
 ├─ Attention
 │    ├─ Wq
 │    ├─ Wk
 │    └─ Wv
 │
 ├─ Feed Forward Network
 │    ├─ Weight
 │    └─ Weight
 │
 └─ 여러 Parameter
 ↓
다음 Token Prediction
 ↓
Loss
 ↓
Backpropagation
 ↓
모든 학습 대상 Parameter의 Gradient 계산
 ↓
Optimizer
 ↓
Parameter Update
```

따라서 Transformer의 구조를 이해하는 것과 신경망의 학습 원리를 이해하는 것은 서로 연결되어 있습니다.

**Transformer는 어떻게 계산하는가**를 설명하고,

**Backpropagation과 Gradient Descent는 그 계산에 사용되는 Parameter를 어떻게 학습하는가**를 설명합니다.

---

# 가장 중요한 한 문장

AI 학습을 이해할 때 가장 중요한 것은 다음 문장입니다.

> **신경망의 학습은 정답과 예측의 차이를 Loss로 계산하고, Backpropagation으로 각 Parameter의 Gradient를 구한 뒤, Optimizer가 Loss를 줄이는 방향으로 Parameter를 반복해서 수정하는 과정이다.**

이를 더 단순하게 표현하면 다음과 같습니다.

```text
예측한다
 ↓
얼마나 틀렸는지 계산한다
 ↓
왜 틀렸는지 Parameter별 영향을 계산한다
 ↓
조금 수정한다
 ↓
다시 예측한다
 ↓
반복한다
```

복잡한 AI 모델도 결국 이 과정에서 출발합니다.

---

# 정리

신경망은 사람이 모든 판단 규칙을 직접 작성하는 방식으로 만들어지지 않습니다.

데이터와 목표를 제공하고 모델이 예측하게 한 뒤, 예측이 틀린 정도를 이용해 내부 Parameter를 반복적으로 수정합니다.

```text
Data
 ↓
Forward
 ↓
Prediction
 ↓
Loss
 ↓
Backward
 ↓
Gradient
 ↓
Optimizer
 ↓
Parameter Update
```

여기서 **Weight와 Bias**는 모델이 학습하는 Parameter이고, **Loss Function**은 현재 모델이 얼마나 잘못하고 있는지 알려줍니다.

**Gradient**는 Parameter를 어느 방향으로 수정해야 Loss가 줄어드는지 알려주고, **Backpropagation**은 복잡한 신경망 전체에서 이 Gradient를 효율적으로 계산합니다.

**Optimizer**는 계산된 Gradient를 이용해 실제 Parameter를 업데이트합니다.

이 과정을 대규모 데이터와 거대한 Transformer에서 수행하는 것이 현대 LLM 학습의 기반입니다.

따라서

```text
신경망
  ↓
Weight
  ↓
Forward
  ↓
Loss
  ↓
Backpropagation
  ↓
Gradient Descent
  ↓
Transformer
  ↓
LLM
```

은 서로 떨어진 개념이 아니라 하나의 연결된 구조입니다.

이 원리를 이해하면 **Pre-training, Fine-tuning, LoRA, Transformer, Parameter, GPU 학습과 추론**이 왜 필요한지도 훨씬 자연스럽게 이해할 수 있습니다.

## 참고 자료

* Ian Goodfellow, Yoshua Bengio, Aaron Courville, [Deep Learning](https://www.deeplearningbook.org/)
* Michael Nielsen, [Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/)
* David E. Rumelhart, Geoffrey E. Hinton, Ronald J. Williams, [Learning representations by back-propagating errors](https://www.nature.com/articles/323533a0)
* Diederik P. Kingma, Jimmy Ba, [Adam: A Method for Stochastic Optimization](https://arxiv.org/abs/1412.6980)
* Ilya Loshchilov, Frank Hutter, [Decoupled Weight Decay Regularization](https://arxiv.org/abs/1711.05101)
* Ashish Vaswani 외, [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
