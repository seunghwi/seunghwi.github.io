---
layout: single
title: "LLM 파라미터와 양자화 완전 이해: 모델 크기를 줄이는 원리와 효과"
categories:
  - "AI"
permalink: /topics/ai/llm-parameters-and-quantization/
tags:
  - "AI"
  - "LLM"
  - "파라미터"
  - "양자화"
  - "모델 최적화"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

LLM을 살펴보면 `7B`, `70B`, `FP16`, `INT8`, `Q4` 같은 표현이 반복해서 등장합니다. 7B 모델은 무엇이 70억 개라는 뜻인지, 4비트 양자화를 하면 왜 파일이 작아지는지, 크기가 줄면 속도도 반드시 빨라지는지는 처음 접할 때 혼동하기 쉽습니다.

**파라미터(parameter)**는 모델이 학습 과정에서 조정한 수치이고, **양자화(quantization)**는 그 수치를 더 적은 비트로 표현해 저장 공간과 계산 비용을 줄이는 방법입니다. 하지만 모델 실행에 필요한 메모리가 전부 파라미터인 것은 아니며, 비트 수를 줄인 만큼 속도가 그대로 빨라지는 것도 아닙니다.

이 글은 파라미터가 신경망 안에서 하는 일, 파라미터 수와 메모리의 관계, 양자화의 수학적 원리와 종류, 실제로 얻는 효과와 잃을 수 있는 품질을 차례대로 설명합니다.

이 글은 **2026년 9월의 공개 논문과 공식 문서**를 기준으로 작성했습니다. 하드웨어와 추론 엔진에 따라 지원하는 데이터 형식과 성능이 다르므로 실제 모델을 배포하기 전에는 사용하는 장비와 런타임의 지원 목록을 확인해야 합니다.
{: .notice--warning}

## 먼저 결론

- 파라미터는 모델이 데이터에서 학습한 숫자이며, 입력을 어떤 출력으로 바꿀지 결정합니다.
- `7B`는 대략 70억 개의 파라미터를 가졌다는 뜻이지 파일이 7GB라는 뜻이 아닙니다.
- 파라미터 저장 공간은 대략 `파라미터 수 × 파라미터당 바이트`로 계산할 수 있습니다.
- 양자화는 많은 연속적인 실수를 제한된 수의 표현 가능한 값에 대응시킵니다.
- FP16 모델을 8비트로 바꾸면 가중치의 이론적 크기는 절반, 4비트로 바꾸면 4분의 1 수준이 됩니다.
- 실제 메모리에는 scale 같은 양자화 메타데이터, KV 캐시, 활성값과 작업 공간이 추가됩니다.
- 양자화는 메모리 사용량과 메모리 대역폭을 줄일 수 있지만 품질 손실과 변환 비용이 생깁니다.
- 실제 속도 향상은 전용 저정밀 연산 커널과 하드웨어가 해당 형식을 효율적으로 지원할 때 얻을 수 있습니다.

## 파라미터란 무엇인가

신경망은 입력 숫자들을 여러 층의 수학 연산에 통과시켜 출력을 만듭니다. 가장 기본적인 선형층은 다음과 같이 표현할 수 있습니다.

~~~text
y = Wx + b

x: 입력 벡터
W: 가중치 행렬
b: 편향 벡터
y: 출력 벡터
~~~

여기서 `W`와 `b`가 **학습 가능한 파라미터**입니다. 처음에는 작은 무작위 값으로 시작할 수 있지만, 학습 데이터에 대한 예측 오차를 줄이도록 반복해서 수정됩니다.

예를 들어 입력 특징이 3개이고 출력 특징이 2개인 선형층이라면 가중치 행렬은 다음처럼 생길 수 있습니다.

~~~text
W = [[ 0.8, -0.2,  0.1],
     [-0.4,  0.7,  0.3]]

b = [0.05, -0.10]
~~~

가중치 6개와 편향 2개를 합쳐 이 층에는 8개의 파라미터가 있습니다. 실제 LLM에서는 이런 행렬의 크기와 층 수가 매우 커져 수십억 개의 파라미터가 만들어집니다.

### 파라미터는 규칙 문장이 아니다

모델 안에 “대한민국의 수도는 서울이다”라는 문장이 하나의 파라미터로 저장되는 것은 아닙니다. 수많은 파라미터가 함께 작동하면서 언어 패턴, 개념 사이의 관계, 문체와 문제 해결 패턴을 **분산된 형태**로 표현합니다.

하나의 파라미터만 보고 그 숫자가 어떤 사실을 담당한다고 해석하기는 어렵습니다. 특정 파라미터를 바꾸면 여러 입력의 결과가 함께 달라질 수 있고, 하나의 지식도 여러 층과 파라미터에 걸쳐 표현될 수 있습니다.

### Transformer의 어디에 파라미터가 있는가

LLM의 Transformer에는 주로 다음 위치에 파라미터가 있습니다.

- **Token embedding**: 토큰 ID를 벡터로 변환하는 표
- **Attention projection**: Query, Key, Value와 출력 벡터를 만드는 행렬
- **Feed-forward network 또는 MLP**: 각 토큰 표현을 확장하고 변환하는 큰 행렬
- **Normalization**: 층의 값 범위를 조절하는 scale과 경우에 따라 bias
- **Output head**: 내부 표현을 다음 토큰의 점수로 바꾸는 행렬

일반적인 Transformer LLM에서는 Attention과 MLP의 큰 행렬이 파라미터 대부분을 차지합니다. 입력 문장의 토큰 수가 늘어도 모델 파라미터 수는 변하지 않습니다. 입력에 따라 달라지는 값은 활성값과 KV 캐시이며 파라미터와 구분해야 합니다.

## 파라미터는 어떻게 학습되는가

언어 모델은 문맥을 보고 다음 토큰의 확률을 예측합니다. 정답 토큰에 낮은 확률을 주면 손실이 커지고, 역전파는 각 파라미터가 손실에 얼마나 영향을 줬는지 gradient를 계산합니다. optimizer는 이 gradient를 이용해 파라미터를 조금씩 변경합니다.

~~~text
학습 데이터 입력
  → 현재 파라미터로 예측
  → 정답과 비교해 손실 계산
  → 역전파로 gradient 계산
  → optimizer가 파라미터 업데이트
  → 다음 batch에서 반복
~~~

학습이 끝나면 추론에서는 파라미터를 일반적으로 고정하고 입력만 통과시킵니다. 파인튜닝은 사전학습된 파라미터 전체나 LoRA 같은 일부 추가 파라미터를 다시 학습해 특정 작업과 행동에 맞추는 과정입니다.

## 파라미터와 하이퍼파라미터의 차이

이름이 비슷하지만 서로 다릅니다.

| 구분 | 파라미터 | 하이퍼파라미터 |
|---|---|---|
| 누가 정하는가 | 학습 과정이 데이터에서 조정 | 개발자가 실험 전에 설정 |
| 예시 | weight, bias, embedding 값 | 학습률, batch size, epoch, layer 수 |
| checkpoint에 저장되는가 | 일반적으로 저장 | 설정 파일이나 학습 기록에 별도 저장 |
| 추론 결과에 미치는 방식 | 계산에 직접 사용 | 모델 구조나 학습 결과에 간접 영향 |

문맥에 따라 `temperature`, `top_p`, 최대 출력 토큰 같은 생성 설정도 추론 하이퍼파라미터라고 부릅니다. 이 값은 응답 선택 방식을 바꾸지만 모델 가중치 자체를 수정하지 않습니다.

## 7B와 70B는 무엇을 뜻하는가

`B`는 billion, 즉 10억을 의미합니다.

- `1B`: 약 10억 개의 파라미터
- `7B`: 약 70억 개의 파라미터
- `70B`: 약 700억 개의 파라미터

파라미터 수가 크면 더 많은 패턴을 표현할 용량이 생길 수 있지만, 크기만으로 모델 품질이 결정되지는 않습니다. 학습 데이터의 양과 품질, 모델 구조, 학습량, tokenizer, 후처리와 평가가 함께 중요합니다. 잘 학습된 작은 모델이 특정 작업에서 더 큰 범용 모델보다 나을 수도 있습니다.

또한 공개 이름의 `7B`는 보통 반올림된 규모입니다. 정확한 파라미터 수는 모델 설정과 구현에서 확인해야 합니다.

## 파라미터 수로 가중치 메모리 계산하기

숫자 하나를 저장하는 데 필요한 비트 수를 알면 가중치의 이론적인 최소 크기를 대략 계산할 수 있습니다.

~~~text
가중치 크기 ≈ 파라미터 수 × 비트 수 ÷ 8
~~~

단순 계산 결과는 다음과 같습니다.

| 모델 규모 | FP32, 4 byte | FP16/BF16, 2 byte | INT8, 1 byte | 4-bit, 0.5 byte |
|---|---:|---:|---:|---:|
| 7B | 약 28GB | 약 14GB | 약 7GB | 약 3.5GB |
| 13B | 약 52GB | 약 26GB | 약 13GB | 약 6.5GB |
| 70B | 약 280GB | 약 140GB | 약 70GB | 약 35GB |

이 표는 **가중치 값만** 계산한 십진 단위의 근사치입니다. 실제 파일과 실행 메모리는 다음 이유로 달라집니다.

- 양자화 그룹마다 scale과 zero point 같은 메타데이터가 추가됩니다.
- tokenizer, 모델 설정과 파일 header가 포함됩니다.
- 일부 layer를 더 높은 정밀도로 남길 수 있습니다.
- 메모리 정렬, padding과 런타임 변환용 buffer가 필요합니다.
- GPU에 로드할 때 파일 형식과 다른 내부 표현으로 풀릴 수 있습니다.

따라서 “4비트 7B는 정확히 3.5GB VRAM이면 실행된다”고 판단하면 안 됩니다.

## 추론 메모리는 파라미터가 전부가 아니다

LLM을 실행할 때 메모리는 크게 다음 요소가 차지합니다.

~~~text
전체 실행 메모리
  = 모델 가중치
  + KV 캐시
  + 중간 활성값
  + 연산 작업 공간
  + 런타임·커널·메모리 할당 여유
~~~

### KV 캐시

Transformer는 새 토큰을 생성할 때 이전 토큰의 Attention Key와 Value를 다시 계산하지 않도록 KV 캐시에 저장합니다. KV 캐시는 batch size, 문맥 길이, layer 수와 KV head 구조에 따라 커집니다.

가중치를 4비트로 줄여도 KV 캐시를 FP16으로 유지하면 긴 문맥과 많은 동시 요청에서 KV 캐시가 메모리의 큰 부분을 차지할 수 있습니다. 그래서 weight quantization과 KV cache quantization은 별도의 선택입니다.

### 활성값과 작업 공간

활성값은 현재 입력이 각 layer를 통과하면서 만들어지는 중간 결과입니다. Flash Attention, fused kernel과 batch 전략에 따라 필요한 공간이 달라집니다. 행렬 곱과 통신을 위한 임시 buffer도 필요합니다.

### 학습 메모리

학습에서는 가중치 외에 gradient, optimizer state와 더 많은 활성값을 저장해야 하므로 추론보다 훨씬 많은 메모리가 필요합니다. FP16 가중치 크기만 보고 같은 GPU에서 전체 파인튜닝까지 가능하다고 생각하면 안 됩니다.

## 숫자의 정밀도란 무엇인가

컴퓨터는 제한된 비트로 숫자를 표현합니다. 비트가 많으면 일반적으로 더 넓은 범위와 더 세밀한 값을 표현할 수 있지만 저장 공간과 데이터 이동량이 늘어납니다.

### 부동소수점

부동소수점은 부호, 지수와 가수 부분을 이용해 매우 크거나 작은 값을 표현합니다.

- **FP32**: 32비트 부동소수점
- **FP16**: 16비트 부동소수점
- **BF16**: FP32와 같은 크기의 지수 영역을 유지하면서 가수 정밀도를 줄인 16비트 형식
- **FP8**: 지수와 가수 배치가 다른 여러 8비트 부동소수점 형식
- **FP4**: 더 낮은 4비트 부동소수점 계열 형식

같은 8비트라도 INT8과 FP8은 표현 방식과 연산 특성이 다릅니다. `8-bit`라는 숫자만 보고 서로 같은 형식으로 취급하면 안 됩니다.

### 정수

INT8은 일반적으로 -128부터 127까지 같은 제한된 정수 값을 표현합니다. 원래의 실수 범위를 scale로 줄여 이 정수 구간에 대응시키고 계산 후 다시 실수 의미로 해석합니다.

INT4는 4비트만 사용하므로 표현 가능한 단계가 훨씬 적습니다. 저장 공간은 더 줄지만 여러 실수가 같은 양자화 값으로 모이면서 오차가 커지기 쉽습니다.

## 양자화란 무엇인가

양자화는 넓고 연속적인 고정밀 값들을 제한된 수의 낮은 정밀도 값에 대응시키는 과정입니다. [Hugging Face 양자화 개념 문서](https://huggingface.co/docs/transformers/quantization/concept_guide)가 설명하는 일반적인 선형 양자화는 scale `S`와 zero point `Z`를 사용해 다음처럼 생각할 수 있습니다.

~~~text
양자화:   q = clamp(round(x / S) + Z)
복원:     x̂ = S × (q - Z)

x : 원래 고정밀 값
q : 저장되는 낮은 정밀도 값
x̂: 다시 계산한 근사값
S : 양자화 간격을 정하는 scale
Z : 실수 0에 대응하는 정수 zero point
~~~

반올림 과정에서 서로 다른 `x`가 같은 `q`로 바뀔 수 있으므로 일반적인 저비트 양자화는 손실 압축입니다. 복원한 `x̂`는 원래 `x`와 정확히 같지 않을 수 있고 그 차이를 **양자화 오차**라고 합니다.

### 간단한 숫자 예제

표현 가능한 정수를 설명하기 쉽게 `-2`, `-1`, `0`, `1`, `2` 다섯 단계로 제한하고 scale을 `0.5`로 정해 보겠습니다.

| 원래 값 x | x / 0.5 | 반올림한 q | 복원값 x̂ | 오차 |
|---:|---:|---:|---:|---:|
| -0.90 | -1.80 | -2 | -1.00 | -0.10 |
| -0.20 | -0.40 | 0 | 0.00 | 0.20 |
| 0.34 | 0.68 | 1 | 0.50 | 0.16 |
| 0.92 | 1.84 | 2 | 1.00 | 0.08 |

표현 단계가 촘촘할수록 오차는 줄지만 더 많은 비트가 필요합니다. 실제 알고리즘은 tensor 전체, channel 또는 작은 group마다 서로 다른 scale을 사용하고 중요한 outlier를 별도로 처리해 오차를 줄입니다.

## 대칭 양자화와 비대칭 양자화

### 대칭 양자화

0을 중심으로 양수와 음수 범위를 같은 크기로 잡습니다. 보통 zero point가 0이라 계산이 단순하고 가중치처럼 0 주변에 분포한 값에 잘 맞을 수 있습니다.

~~~text
실수 범위: -1.2 ───── 0 ───── 1.2
정수 범위: -127 ───── 0 ───── 127
~~~

### 비대칭 양자화

실제 최소값과 최대값에 맞춰 범위를 이동하고 0에 대응하는 zero point를 둡니다. 값이 한쪽으로 치우친 분포를 더 효율적으로 표현할 수 있지만 계산과 메타데이터가 조금 복잡해집니다.

어느 방식이 항상 우수한 것은 아닙니다. quantization backend, 연산 커널과 대상 tensor의 분포에 따라 결정합니다.

## Granularity: scale을 어디까지 공유하는가

양자화의 품질과 효율은 scale을 계산하는 단위에도 크게 영향을 받습니다.

| 방식 | scale 공유 범위 | 특징 |
|---|---|---|
| Per-tensor | tensor 전체 | 가장 단순하지만 outlier 하나가 전체 해상도를 낮출 수 있음 |
| Per-channel | 출력 또는 입력 channel별 | 분포 차이를 반영해 더 정확하지만 scale 수가 증가 |
| Per-group/block | 일정 개수의 weight 묶음별 | 4비트 weight-only 양자화에서 자주 쓰는 절충안 |
| Per-token | activation의 token별 | 입력마다 달라지는 activation 범위 대응에 활용 가능 |

예를 들어 group size가 작으면 각 그룹의 값 범위에 scale을 더 세밀하게 맞출 수 있어 품질에 유리할 수 있지만 scale과 zero point의 비중이 커지고 kernel 효율이 달라질 수 있습니다.

그래서 `4-bit 모델`이라는 정보만으로 크기와 품질을 정확히 비교할 수 없습니다. 알고리즘, group size, 대칭 여부, 일부 layer의 정밀도와 계산 dtype까지 확인해야 합니다.

## 무엇을 양자화하는가

### Weight-only quantization

가중치만 INT8이나 INT4로 저장하고 활성값과 실제 누산 계산은 FP16/BF16 같은 더 높은 정밀도를 사용합니다. `W4A16`은 보통 **weight 4비트, activation 16비트**를 의미합니다.

가중치 메모리와 메모리 대역폭을 크게 줄이면서 activation 양자화의 어려움을 피할 수 있어 LLM 로컬 실행과 GPU 추론에서 널리 사용됩니다. 연산 전에 필요한 블록을 dequantize하거나 저비트 전용 kernel에서 직접 처리합니다.

### Weight-and-activation quantization

`W8A8`처럼 weight와 activation을 모두 8비트로 계산합니다. 행렬 곱 자체를 효율적인 저정밀 연산으로 바꿀 수 있어 적절한 하드웨어에서 높은 처리량을 기대할 수 있습니다.

LLM activation에는 일부 channel에 매우 큰 outlier가 나타날 수 있어 단순 INT8 변환이 어렵습니다. [LLM.int8()](https://arxiv.org/abs/2208.07339)은 대부분을 8비트로 계산하면서 outlier 차원을 더 높은 정밀도로 분리합니다. [SmoothQuant](https://arxiv.org/abs/2211.10438)은 activation의 양자화 난도를 수학적으로 동등한 scale 변환을 통해 weight 쪽으로 옮겨 W8A8 양자화를 쉽게 만듭니다.

### KV cache quantization

생성 중 저장하는 Key와 Value를 낮은 정밀도로 줄입니다. 긴 문맥이나 큰 batch에서 메모리 절감 효과가 크지만 Attention 결과에 누적되는 오차와 kernel 지원을 확인해야 합니다. weight-only 모델이라고 KV 캐시까지 자동으로 4비트가 되는 것은 아닙니다.

### 일부만 높은 정밀도로 남기는 혼합 정밀도

embedding, output head, normalization 또는 outlier channel처럼 민감한 부분을 FP16/BF16로 남기고 나머지만 낮은 비트로 바꿀 수 있습니다. 전체 크기는 조금 커지지만 품질 저하를 줄이는 현실적인 절충입니다.

## 언제 양자화하는가

### PTQ: 학습 후 양자화

**Post-Training Quantization(PTQ)**은 학습이 끝난 모델을 추가 전체 학습 없이 변환합니다. 빠르고 기존 checkpoint에 적용하기 쉬워 추론용 LLM 양자화에서 널리 사용됩니다.

방법에 따라 대표 입력을 담은 calibration dataset으로 activation이나 weight 통계를 측정합니다. calibration 데이터가 실제 사용 분포와 너무 다르면 scale과 중요도 판단이 부정확해져 품질이 떨어질 수 있습니다.

- **RTN(Round-to-Nearest)**: 정한 scale에 따라 단순 반올림
- **GPTQ**: layer별로 양자화 오차를 보정하는 대표적인 weight-only PTQ 방식
- **AWQ**: activation 통계로 중요한 weight channel을 찾아 보호하는 weight-only 방식
- **SmoothQuant**: activation outlier 문제를 weight로 이동시켜 W8A8을 가능하게 하는 방식

[GPTQ 논문](https://arxiv.org/abs/2210.17323)과 [AWQ 논문](https://arxiv.org/abs/2306.00978)은 특히 낮은 비트의 LLM weight를 학습 후 변환하는 대표적인 접근입니다. 이름이 비슷한 파일 형식과 알고리즘을 구분하고, 사용하는 엔진이 해당 checkpoint를 위한 kernel을 지원하는지 확인해야 합니다.

### QAT: 양자화 인식 학습

**Quantization-Aware Training(QAT)**은 학습 중에 반올림과 clipping으로 생기는 양자화 효과를 모의 연산으로 반영합니다. 모델이 오차에 적응할 기회를 주므로 낮은 비트에서 PTQ보다 품질을 잘 보존할 가능성이 있지만 학습 데이터, 계산 자원과 구현 복잡성이 더 필요합니다.

[정수 연산 신경망 양자화 연구](https://openaccess.thecvf.com/content_cvpr_2018/html/Jacob_Quantization_and_Training_CVPR_2018_paper.html)는 학습 중 fake quantization으로 양자화 효과를 모사하고 정수 추론을 가능하게 하는 기본 구조를 설명합니다.

QAT를 했다고 항상 더 좋은 것은 아닙니다. 높은 품질의 원본 학습 데이터가 없거나 이미 충분히 정확한 8비트 PTQ가 가능하면 추가 비용의 가치가 작을 수 있습니다.

## 양자화가 주는 효과

### 1. 모델 파일과 메모리가 작아진다

가장 직접적인 효과입니다. 같은 파라미터 수를 더 적은 비트로 저장하므로 디스크, RAM과 VRAM 사용량이 줄어듭니다.

- 기존 장비에 더 큰 모델을 올릴 수 있습니다.
- 한 GPU에 더 많은 model replica나 adapter를 배치할 수 있습니다.
- 여러 GPU가 필요했던 모델이 더 적은 GPU에서 동작할 수 있습니다.
- 모바일과 edge device에서 실행 가능성이 커집니다.

다만 앞서 본 것처럼 scale 메타데이터와 런타임 메모리가 있으므로 정확히 비트 비율만큼 전체 메모리가 줄지는 않습니다.

### 2. 메모리 대역폭 부담이 줄어든다

LLM의 token-by-token decode는 매 단계에서 큰 weight를 반복해서 읽기 때문에 메모리 대역폭에 제한되는 경우가 많습니다. weight 크기가 줄면 같은 시간에 더 많은 값을 메모리에서 연산 장치로 전달할 수 있습니다.

이 때문에 weight-only 4비트도 계산 자체는 높은 정밀도로 수행하면서 decode latency와 throughput을 개선할 수 있습니다. 효과의 크기는 batch, 모델 크기, 메모리 대역폭, dequantization 비용과 kernel 구현에 따라 달라집니다.

### 3. 저정밀 연산 장치를 사용할 수 있다

하드웨어가 INT8, FP8, INT4 또는 FP4 연산을 빠르게 지원하고 런타임에 최적화된 kernel이 있다면 행렬 곱 처리량을 높일 수 있습니다. W8A8처럼 activation까지 낮은 정밀도로 계산하는 방식은 이 이점을 직접 활용합니다.

### 4. 비용과 전력 사용을 줄일 수 있다

필요한 GPU 수와 데이터 이동량, 요청당 계산 시간이 줄면 서비스 비용과 전력 사용도 낮아질 수 있습니다. 같은 장비에서 동시 요청을 더 많이 처리할 수 있다면 처리량당 비용이 좋아집니다.

다만 양자화 변환, 품질 평가, 여러 checkpoint 보관과 호환성 관리 비용도 포함해 전체 운영 비용을 비교해야 합니다.

## 양자화한다고 항상 빨라지는 것은 아니다

모델 파일이 작아지는 효과는 비교적 확실하지만 속도 향상은 조건부입니다.

1. 하드웨어가 그 데이터 형식을 빠르게 계산해야 합니다.
2. 추론 엔진에 최적화된 kernel이 있어야 합니다.
3. 매 연산마다 높은 정밀도로 풀어내는 비용이 절감 효과보다 작아야 합니다.
4. workload가 weight memory bandwidth 또는 행렬 곱에 실제로 병목이어야 합니다.
5. batch size와 입력·출력 길이가 kernel 특성에 맞아야 합니다.

지원하지 않는 형식은 CPU에서 변환하거나 일반 연산으로 우회해 오히려 느려질 수 있습니다. 작은 batch에서는 kernel 실행과 dequantization overhead가 상대적으로 크게 보일 수 있습니다. 입력이 매우 긴 prefill은 decode와 다른 병목을 가질 수 있습니다.

따라서 모델 카드의 “4-bit” 표시만 보고 성능을 예상하지 말고 실제 하드웨어, 엔진, 문맥 길이와 동시 요청 수로 benchmark해야 합니다.

## 품질은 왜 떨어질 수 있는가

양자화는 여러 고정밀 값을 같은 낮은 정밀도 값으로 합칩니다. 작은 오차가 layer를 거치며 결과 분포를 바꿀 수 있고, 다음 토큰 확률이 비슷한 상황에서는 선택 토큰 자체가 달라질 수 있습니다.

품질 손실에 영향을 주는 요소는 다음과 같습니다.

- 비트 수: 일반적으로 낮을수록 표현 단계가 줄어듭니다.
- 모델 크기와 구조: 같은 방식에도 민감도가 다릅니다.
- layer와 channel별 outlier 분포
- group size와 scale 계산 방식
- calibration 데이터의 대표성
- 양자화 대상: weight, activation, KV cache
- 평가 작업: 자연어, 코드, 수학, 장문과 희귀 언어
- 추론 엔진의 kernel과 수치 구현

8비트에서 차이가 거의 없던 모델도 4비트에서 특정 작업이 약해질 수 있고, 평균 benchmark는 유지되지만 드문 형식이나 긴 문맥에서 실패할 수도 있습니다. perplexity 하나만 보지 말고 실제 업무 평가셋, 형식 준수, 사실성, 안전성과 latency를 함께 비교해야 합니다.

양자화 후 답이 달라졌다고 모두 품질 저하는 아닙니다. 생성 모델은 sampling 설정과 runtime에 따라 원래도 출력이 달라질 수 있습니다. 동일한 prompt, seed가 지원되는 경우 같은 seed, decoding 설정과 충분한 평가 표본으로 비교해야 합니다.
{: .notice--info}

## 비트 수만 같아도 결과가 다른 이유

두 모델 파일이 모두 `Q4`라고 적혀 있어도 다음 조건이 다를 수 있습니다.

- uniform integer인지 NF4 같은 비균일 codebook인지
- symmetric인지 asymmetric인지
- per-channel인지 group-wise인지
- group size가 몇 개인지
- 중요한 layer를 더 높은 정밀도로 남겼는지
- weight만 줄였는지 KV cache도 줄였는지
- GPTQ, AWQ 등 어떤 calibration·보정 알고리즘을 썼는지
- 어떤 tensor packing과 kernel을 사용하는지

그래서 모델을 선택할 때는 단순 파일 크기 외에 quantization config, 원본 모델 hash, calibration 정보, 지원 runtime과 품질 평가를 확인해야 합니다.

## 자주 보는 이름 정리

### GPTQ

학습이 끝난 Transformer의 weight를 layer 단위로 낮은 비트에 맞추면서 출력 오차를 줄이는 대표적인 PTQ 알고리즘입니다. 주로 4비트 weight-only 배포에서 접하게 됩니다.

### AWQ

모든 weight가 같은 중요도를 갖지 않는다는 관점에서 activation 통계를 이용해 중요한 weight channel을 보호합니다. 별도 전체 역전파 없이 calibration을 사용하는 weight-only PTQ 방식입니다.

### SmoothQuant

LLM activation의 outlier 때문에 어려운 W8A8 양자화를 위해 activation의 크기 변동 일부를 weight 쪽으로 옮기는 동등 변환을 사용합니다. 적절한 INT8 kernel에서 weight와 activation을 모두 낮은 정밀도로 계산하는 데 초점을 둡니다.

### bitsandbytes

8비트와 4비트 모델 로딩, 계산과 QLoRA 학습에 사용되는 library/backend입니다. 알고리즘 이름, 저장 형식과 실행 library는 같은 개념이 아니므로 구분해야 합니다. 현재 지원 방식은 [Transformers 양자화 문서](https://huggingface.co/docs/transformers/quantization/concept_guide)에서 확인할 수 있습니다.

### GGUF와 Q4 계열

GGUF는 모델 tensor와 metadata를 담는 파일 형식입니다. 그 안에 여러 양자화 유형이 들어갈 수 있으며 `Q4_K` 같은 이름은 특정 구현의 block quantization 방식을 나타냅니다. GGUF 자체가 단 하나의 양자화 알고리즘을 뜻하지는 않습니다.

### FP8과 FP4

낮은 비트의 부동소수점 형식입니다. INT 계열과 동적 범위와 정밀도 분배가 다르며 특정 GPU 가속 기능과 함께 사용됩니다. 같은 이름 아래에도 지수·가수 배치와 scale 방식이 다른 variant가 있으므로 하드웨어와 framework 지원을 함께 확인합니다.

## 양자화와 파인튜닝의 관계

일반적인 추론용 PTQ는 학습이 끝난 모델 weight를 줄여 실행하는 과정입니다. 파인튜닝은 데이터로 모델의 행동을 바꾸는 학습 과정이므로 목적이 다릅니다.

[QLoRA](https://arxiv.org/abs/2305.14314)는 동결된 기본 모델을 4비트 NF4로 저장하고 그 모델을 통과해 작은 LoRA adapter를 학습하는 방법입니다. 기본 모델의 모든 4비트 값을 직접 업데이트하는 일반적인 full fine-tuning과는 다릅니다.

~~~text
4비트 추론용 모델
  → 목적: 메모리와 추론 비용 절감

QLoRA
  → 동결된 4비트 기본 모델 + 학습 가능한 LoRA adapter
  → 목적: 제한된 메모리에서 파라미터 효율적 파인튜닝
~~~

학습이 끝난 LoRA adapter를 기본 모델과 합친 뒤 다시 배포용 양자화를 수행할 수도 있습니다. 순서와 지원 범위는 도구마다 다르므로 checkpoint를 무작정 변환하지 말고 권장 pipeline을 따라야 합니다.

## 직접 이해해 보는 간단한 Python 예제

다음 코드는 외부 library 없이 대칭 INT8 양자화의 핵심만 단순화해 보여줍니다. 실제 LLM 양자화 library를 대신하는 코드는 아닙니다.

~~~python
weights = [-1.20, -0.70, -0.05, 0.34, 0.92]

# signed INT8의 양수 최대값
qmax = 127

# 절댓값이 가장 큰 값을 INT8 범위 끝에 맞춘다.
scale = max(abs(value) for value in weights) / qmax

# 양자화: 실수를 INT8 범위의 정수로 변환한다.
quantized = [
    max(-qmax, min(qmax, round(value / scale)))
    for value in weights
]

# 복원: 계산에 사용할 근삿값을 만든다.
restored = [value * scale for value in quantized]

for original, integer, approximate in zip(
    weights, quantized, restored
):
    print(
        f"원본={original:>5.2f}, "
        f"정수={integer:>4}, "
        f"복원={approximate:>7.4f}"
    )
~~~

출력은 대략 다음과 같습니다.

~~~text
원본=-1.20, 정수=-127, 복원=-1.2000
원본=-0.70, 정수= -74, 복원=-0.6992
원본=-0.05, 정수=  -5, 복원=-0.0472
원본= 0.34, 정수=  36, 복원= 0.3402
원본= 0.92, 정수=  97, 복원= 0.9165
~~~

원래 Python 실수 다섯 개가 실제 메모리에서 이 코드만으로 압축되는 것은 아닙니다. 예제는 수학적 대응만 보여줍니다. 실제 저장 공간을 줄이려면 정수를 INT8 buffer로 packing하고 scale과 tensor 구조를 별도로 저장해야 합니다.

## 어떤 정밀도를 선택해야 하는가

### FP16 또는 BF16

- 품질 저하 위험을 최소화하고 싶은 경우
- 충분한 GPU 메모리가 있는 경우
- 학습이나 범용성이 중요한 기준 모델
- 양자화 kernel이 없는 하드웨어

### 8비트

- 비교적 보수적으로 메모리를 줄이고 싶은 경우
- W8A8을 지원하는 서버 GPU에서 높은 처리량이 필요한 경우
- 4비트 품질 저하가 허용되지 않는 작업

### 4비트

- 로컬 PC와 edge device에서 큰 모델을 실행해야 하는 경우
- GPU 메모리가 핵심 제약인 경우
- weight bandwidth가 decode 병목인 경우
- 실제 평가에서 품질이 허용 범위에 들어오는 경우

### 3비트 이하

- 메모리 제약이 매우 강한 실험적·특수 환경
- 전용 kernel과 지원 runtime이 있는 경우
- 더 큰 품질 손실과 호환성 제한을 충분히 평가할 수 있는 경우

비트 수를 먼저 결정하기보다 목표 품질, 장비, latency, throughput, 문맥 길이와 동시 요청 수를 정의한 뒤 후보를 비교하는 편이 좋습니다.

## 실전 선택과 검증 순서

1. **기준 모델을 고정합니다.** 원본 checkpoint와 tokenizer 버전을 기록합니다.
2. **실제 workload를 만듭니다.** 입력·출력 길이, batch와 동시 요청 분포를 반영합니다.
3. **품질 기준선을 측정합니다.** FP16/BF16 모델의 작업별 결과를 저장합니다.
4. **하드웨어 지원을 확인합니다.** runtime과 kernel이 후보 형식을 지원하는지 봅니다.
5. **8비트와 4비트를 비교합니다.** 파일 크기뿐 아니라 실제 peak memory를 측정합니다.
6. **latency와 throughput을 따로 측정합니다.** 첫 토큰 시간, token 간 지연과 동시 처리량을 기록합니다.
7. **업무 품질을 평가합니다.** 일반 benchmark와 실제 질문, 긴 문맥, 코드와 안전 사례를 함께 봅니다.
8. **운영 조건을 확인합니다.** cold start, model loading 시간, 여러 GPU와 adapter 호환성을 시험합니다.
9. **변환 정보를 기록합니다.** 알고리즘, bit, group size, calibration data와 도구 버전을 남깁니다.

## 흔한 오해

### 파라미터 수가 크면 항상 더 똑똑하다

파라미터 수는 모델 용량을 나타내는 한 요소입니다. 데이터와 학습 품질, 구조와 목적이 다르면 크기만으로 성능을 비교할 수 없습니다.

### 4비트 모델은 원본보다 정확히 4배 작다

FP16 weight 값만 비교하면 이론적으로 4분의 1이지만 scale, zero point, 일부 고정밀 tensor와 파일 metadata 때문에 실제 비율은 달라집니다.

### 4비트면 FP16보다 정확히 4배 빠르다

속도는 메모리 대역폭, 저비트 연산 지원, kernel, dequantization, batch와 workload에 달렸습니다. 크기 감소와 속도 향상은 분리해서 측정해야 합니다.

### 양자화하면 모델의 지식이 전부 사라진다

적절한 방법과 비트 수에서는 대부분의 능력을 유지할 수 있지만 일부 품질 손실 가능성은 있습니다. 손실 정도는 모델과 작업마다 다르므로 평가 없이 단정할 수 없습니다.

### 모든 4비트 파일은 서로 바꿔 쓸 수 있다

저장 container, tensor layout, 양자화 방식과 kernel이 다르면 호환되지 않습니다. 모델 파일과 실행 엔진의 지원 조합을 확인해야 합니다.

## 정리

파라미터는 모델이 학습한 수치이며 Transformer의 embedding, Attention, MLP와 출력층에서 입력을 변환합니다. `7B`, `70B`는 이 수치의 개수를 나타내고 실제 저장 크기는 각 수치를 몇 비트로 표현하는지에 따라 달라집니다.

양자화는 고정밀 값을 제한된 낮은 정밀도 단계로 대응시켜 모델을 압축합니다. 이를 통해 모델 파일, RAM·VRAM과 메모리 대역폭 부담을 줄이고 적절한 하드웨어에서는 추론 속도와 처리량도 개선할 수 있습니다.

그 대가로 반올림 오차, 품질 저하, calibration과 kernel 의존성이 생깁니다. 따라서 가장 낮은 비트가 가장 좋은 선택은 아닙니다. **목표 품질을 만족하는 범위에서 실제 장비와 workload에 가장 효율적인 형식**을 찾는 것이 양자화의 핵심입니다.

## 참고 자료

- Hugging Face, [Quantization concepts](https://huggingface.co/docs/transformers/quantization/concept_guide)
- Benoit Jacob 외, [Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference](https://openaccess.thecvf.com/content_cvpr_2018/html/Jacob_Quantization_and_Training_CVPR_2018_paper.html)
- Tim Dettmers 외, [LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale](https://arxiv.org/abs/2208.07339)
- Elias Frantar 외, [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers](https://arxiv.org/abs/2210.17323)
- Guangxuan Xiao 외, [SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models](https://arxiv.org/abs/2211.10438)
- Ji Lin 외, [AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration](https://arxiv.org/abs/2306.00978)
- Tim Dettmers 외, [QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314)
- NVIDIA TensorRT, [Quantization schemes](https://docs.nvidia.com/deeplearning/tensorrt/latest/inference-library/quantized-types-schemes.html)
