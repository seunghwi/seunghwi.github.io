---
layout: single
title: "AI 학습 시뮬레이터 설명"
categories:
  - "AI"
permalink: /topics/ai/learning-simulator-explain/
tags:
  - "AI"
  - "LLM"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

# AI 학습의 기본 원리와 직접 만들어보는 학습 시뮬레이터

AI를 처음 접하면 거대한 신경망이나 ChatGPT 같은 LLM부터 떠올리기 쉽습니다.

하지만 AI 학습의 가장 기본적인 구조는 생각보다 단순합니다.

이 글에서는 아주 작은 모델이 다음 관계를 스스로 학습하는 과정을 직접 구현합니다.

```text
x = 1 → y = 2
x = 2 → y = 4
x = 3 → y = 6
x = 4 → y = 8
```

사람은 이 데이터를 보면 금방 `y = 2x`라는 관계를 알아차릴 수 있습니다.

하지만 프로그램에는 `y = 2x`라는 공식을 직접 알려주지 않습니다.

대신 프로그램이 **예측 → 오차 계산 → 가중치 수정 → 다시 예측**을 반복하면서 스스로 `w ≈ 2`를 찾아가게 만듭니다.

---

## 1. AI 학습의 전체 흐름

이번 프로그램의 학습 과정은 다음과 같습니다.

```text
학습 데이터
    ↓
현재 가중치로 예측
    ↓
정답과 비교
    ↓
Loss 계산
    ↓
Gradient 계산
    ↓
가중치 수정
    ↓
다시 예측
    ↓
반복
```

핵심 개념은 다음과 같습니다.

| 프로그램의 요소 | AI 용어 |
|---|---|
| `data` | Training Data |
| `x` | Input |
| `y` | Label / Target |
| `w` | Weight / Parameter |
| `predict()` | Model / Inference |
| `loss()` | Loss Function |
| `gradient()` | Gradient |
| `learningRate` | Learning Rate |
| `w = w - lr * g` | Gradient Descent |
| `epoch` | Epoch |

---

# 2. 학습 데이터

프로그램에서 사용하는 학습 데이터는 다음과 같습니다.

```javascript
const data = [
  {x:1, y:2},
  {x:2, y:4},
  {x:3, y:6},
  {x:4, y:8}
];
```

AI가 받는 정보는 이것뿐입니다.

```text
입력 x    정답 y

1    →    2
2    →    4
3    →    6
4    →    8
```

코드 어디에도 다음과 같이 정답 공식을 직접 넣지 않습니다.

```javascript
w = 2;
```

프로그램은 학습을 통해 결과적으로 `w ≈ 2`를 찾아야 합니다.

---

# 3. 가중치 Weight란?

프로그램은 처음에 다음과 같이 가중치를 하나 가지고 시작합니다.

```javascript
let w = 0.3;
```

이번 모델에서 `w`는 AI가 학습해야 하는 **Parameter(매개변수)**입니다.

모델의 계산식은 아주 단순합니다.

```text
예측값 = x × w
```

처음에는,

```text
w = 0.3
```

이므로 `x = 3`을 입력하면,

```text
예측 = 3 × 0.3
     = 0.9
```

가 됩니다.

하지만 실제 정답은,

```text
6
```

입니다.

즉 현재 가중치는 좋은 값이 아닙니다.

AI 학습의 목적은 이 `w`를 조금씩 수정하여 예측을 정답에 가깝게 만드는 것입니다.

---

# 4. 예측 함수

프로그램의 예측 부분은 매우 간단합니다.

```javascript
function predict(x) {
  return x * w;
}
```

입력값 `x`에 현재 가중치 `w`를 곱합니다.

```text
입력 x
  ↓
가중치 w와 곱하기
  ↓
예측값 ŷ
```

예를 들어,

```text
x = 4
w = 0.3
```

이라면,

```text
4 × 0.3 = 1.2
```

를 예측합니다.

이 계산이 이 작은 프로그램의 **모델(Model)**입니다.

실제 신경망에서는 이 계산이 훨씬 많아집니다.

```text
입력1 × 가중치1
+
입력2 × 가중치2
+
입력3 × 가중치3
+
Bias
    ↓
활성화 함수
    ↓
출력
```

즉 구조는 복잡해져도 입력과 가중치를 이용해 출력을 계산한다는 기본 개념은 이어집니다.

---

# 5. Loss란?

AI가 학습하려면 현재 예측이 얼마나 틀렸는지를 알아야 합니다.

이를 숫자로 나타낸 것이 **Loss(손실)**입니다.

프로그램에서는 다음 함수를 사용합니다.

```javascript
function loss() {
  let total = 0;

  for (const d of data) {
    const e = predict(d.x) - d.y;
    total += e * e;
  }

  return total / data.length;
}
```

먼저,

```javascript
const e = predict(d.x) - d.y;
```

를 통해 예측값과 정답의 차이를 구합니다.

예를 들어,

```text
예측 = 0.9
정답 = 6
```

이라면,

```text
오차 = 0.9 - 6
     = -5.1
```

입니다.

---

# 6. 오차를 왜 제곱할까?

프로그램은 오차를 그대로 더하지 않고,

```javascript
e * e
```

를 계산합니다.

오차가 다음처럼 존재한다고 생각해봅시다.

```text
+5
-5
```

그대로 더하면,

```text
+5 + (-5) = 0
```

이 됩니다.

하지만 두 예측 모두 실제로는 크게 틀렸습니다.

그래서 오차를 제곱합니다.

```text
5²     = 25
(-5)²  = 25
```

그리고 모든 데이터의 제곱 오차 평균을 구합니다.

이를 **MSE(Mean Squared Error, 평균제곱오차)**라고 합니다.

```text
Loss가 큼
→ 예측이 많이 틀림

Loss가 작음
→ 예측이 정답에 가까움
```

따라서 학습의 목표는 결국,

```text
Loss를 최대한 작게 만드는 것
```

이라고 볼 수 있습니다.

---

# 7. Gradient란?

이제 중요한 문제가 하나 생깁니다.

현재 `w = 0.3`이 잘못되었다는 것은 Loss를 통해 알았습니다.

그런데 `w`를 올려야 할까요?

아니면 내려야 할까요?

그리고 얼마나 움직여야 할까요?

이를 알아내는 데 사용하는 것이 **Gradient(기울기)**입니다.

프로그램의 코드는 다음과 같습니다.

```javascript
function gradient() {
  let total = 0;

  for (const d of data) {
    const e = predict(d.x) - d.y;
    total += 2 * e * d.x;
  }

  return total / data.length;
}
```

핵심 부분은,

```javascript
2 * e * d.x
```

입니다.

이 값은 현재 가중치 `w`를 변화시켰을 때 Loss가 어느 방향으로 얼마나 변하는지를 알려줍니다.

쉽게 생각하면,

```text
현재 위치에서
Loss가 작아지는 방향은 어디인가?
```

를 알아내는 것입니다.

---

# 8. 왜 미분이 AI 학습에 등장할까?

Loss를 산의 높이라고 생각하면 이해하기 쉽습니다.

```text
Loss
 ^
 |        /\
 |       /  \
 |      /    \
 |_____/______\________> w
```

우리가 원하는 것은 가장 낮은 곳입니다.

```text
Loss 최소
```

현재 위치에서 경사를 보면 어느 방향으로 내려가야 하는지 알 수 있습니다.

이 경사를 계산하는 것이 미분이고, 여러 변수가 있을 때 이를 Gradient라고 부릅니다.

```text
Gradient > 0
→ 반대 방향으로 w를 감소

Gradient < 0
→ 반대 방향으로 w를 증가
```

---

# 9. Gradient Descent

Gradient를 계산했다면 실제로 가중치를 수정합니다.

프로그램에서 가장 중요한 코드 중 하나입니다.

```javascript
w = w - lr * g;
```

여기서,

```text
w  = 현재 가중치
lr = Learning Rate
g  = Gradient
```

입니다.

예를 들어,

```text
w = 0.3
gradient = -20
learning rate = 0.01
```

이라면,

```text
w = 0.3 - (0.01 × -20)
```

이므로,

```text
w = 0.5
```

가 됩니다.

다시 계산하면 가중치가 계속 수정됩니다.

```text
0.3
 ↓
0.5
 ↓
0.8
 ↓
1.2
 ↓
1.7
 ↓
1.95
 ↓
2.00
```

이처럼 Loss가 작아지는 방향으로 조금씩 이동하는 방법을 **Gradient Descent(경사하강법)**라고 합니다.

---

# 10. Learning Rate는 무엇일까?

Learning Rate는 한 번 학습할 때 가중치를 얼마나 크게 움직일지를 결정합니다.

```javascript
const lr = Number(
  document.getElementById('learningRate').value
);
```

그리고,

```javascript
w = w - lr * g;
```

에 사용됩니다.

학습률이 너무 작으면,

```text
0.300
0.301
0.302
0.303
...
```

처럼 학습이 매우 느릴 수 있습니다.

반대로 너무 크면 최적점을 지나칠 수 있습니다.

```text
1.0
 ↓
2.8
 ↓
0.4
 ↓
3.5
 ↓
-1.0
```

심한 경우 값이 계속 커지면서 학습이 발산할 수도 있습니다.

따라서 Learning Rate는 AI 학습에서 중요한 **Hyperparameter**입니다.

시뮬레이터에서 `0.01`, `0.1`, `0.5` 등으로 직접 바꾸어 보면 차이를 쉽게 확인할 수 있습니다.

---

# 11. Epoch란?

프로그램에는 다음 코드가 있습니다.

```javascript
epoch++;
```

Epoch는 학습 데이터 전체를 이용한 학습 반복 횟수를 나타냅니다.

예를 들어,

```text
Epoch 1
w = 0.55

Epoch 10
w = 1.52

Epoch 50
w = 1.97

Epoch 100
w = 1.999
```

처럼 반복할수록 적절한 가중치에 가까워질 수 있습니다.

다만 실제 AI에서는 Epoch를 무조건 많이 늘린다고 항상 좋은 것은 아닙니다.

---

# 12. 한 번의 학습에서 실제로 일어나는 일

시뮬레이터의 핵심 함수는 다음과 같습니다.

```javascript
function trainOne() {
  const lr = Number(
    document.getElementById('learningRate').value
  );

  const g = gradient();

  const oldW = w;

  w = w - lr * g;

  epoch++;

  const l = loss();

  lossHistory.push(l);

  update();
}
```

이를 사람이 이해하기 쉬운 순서로 바꾸면 다음과 같습니다.

```text
현재 Weight 확인
      ↓
현재 Weight로 예측
      ↓
정답과 비교
      ↓
Loss 계산
      ↓
Gradient 계산
      ↓
Weight 수정
      ↓
Epoch 증가
      ↓
새로운 Weight로 다시 계산
```

그리고 이 과정을 반복합니다.

---

# 13. AI 학습의 핵심만 코드로 줄이면

화면 출력과 애니메이션 코드를 모두 제외하면 핵심 개념은 매우 작습니다.

```javascript
const prediction = x * w;
const error = prediction - y;
const g = gradient();

w = w - learningRate * g;
```

즉,

```text
예측한다.
 ↓
얼마나 틀렸는지 확인한다.
 ↓
어느 방향으로 수정할지 계산한다.
 ↓
가중치를 수정한다.
```

이 과정을 반복하는 것입니다.

---

# 14. 왜 이것을 AI 학습이라고 할 수 있을까?

중요한 점은 프로그램에 다음 정답을 직접 넣지 않았다는 것입니다.

```text
w = 2
```

프로그램이 알고 있는 것은 학습 데이터뿐입니다.

```text
1 → 2
2 → 4
3 → 6
4 → 8
```

처음에는,

```text
w = 0.3
```

에서 시작하지만 데이터에 대한 Loss를 줄이는 방향으로 계속 수정합니다.

결국,

```text
w ≈ 2
```

를 찾아냅니다.

즉 사람이 최종 Parameter 값을 직접 작성한 것이 아니라 **데이터를 이용한 최적화 과정을 통해 Parameter가 결정된 것**입니다.

---

# 15. 실제 신경망과 무엇이 다를까?

현재 프로그램에는 학습할 가중치가 하나뿐입니다.

```text
x
 ↓
w
 ↓
y
```

신경망에서는 입력과 가중치가 많아집니다.

```text
x1 ── w1 ─┐
x2 ── w2 ─┼→ 뉴런 → 출력
x3 ── w3 ─┘
```

여러 뉴런을 연결하면,

```text
입력층

○ ○ ○
│ │ │
↓ ↓ ↓

은닉층

○ ○ ○ ○
│ │ │ │
↓ ↓ ↓ ↓

출력층

○
```

같은 구조가 됩니다.

각 연결에는 서로 다른 가중치가 존재합니다.

학습의 목적은 수많은 가중치를 조정해서 Loss를 줄이는 것입니다.

---

# 16. Bias는 무엇일까?

현재 예제는 단순화를 위해,

```text
y = wx
```

만 사용합니다.

조금 더 일반적인 모델은,

```text
y = wx + b
```

입니다.

여기서 `b`가 **Bias(편향)**입니다.

예를 들어,

```text
y = 2x + 3
```

을 학습하려면 가중치 `w`뿐 아니라 Bias `b`도 학습해야 합니다.

```text
학습 전

w = 0.3
b = 0.1

        ↓ 학습

w ≈ 2
b ≈ 3
```

이렇게 학습할 Parameter가 하나 더 늘어납니다.

---

# 17. 실제 신경망에서는 Backpropagation을 사용한다

현재 예제는 가중치가 하나라 Gradient 계산이 매우 간단합니다.

하지만 신경망에는 수많은 가중치가 있습니다.

```text
입력
 ↓
Layer 1
 ↓
Layer 2
 ↓
Layer 3
 ↓
출력
 ↓
Loss
```

각 가중치가 Loss에 얼마나 영향을 미쳤는지 계산해야 합니다.

이때 사용하는 핵심 알고리즘이 **Backpropagation(역전파)**입니다.

개념적으로는,

```text
Forward

입력
 ↓
신경망
 ↓
예측
 ↓
Loss

Backward

Loss
 ↑
각 Weight가 Loss에 미친 영향 계산
 ↑
Gradient 계산
 ↑
Weight 수정
```

입니다.

현재 프로그램의 `gradient()`는 이런 개념을 아주 단순한 형태로 직접 계산한 것입니다.

---

# 18. ChatGPT 같은 LLM과의 관계

ChatGPT 같은 LLM은 지금 프로그램보다 비교할 수 없을 정도로 복잡합니다.

하지만 큰 흐름에서는 연결되는 부분이 있습니다.

현재 프로그램:

```text
숫자 입력
 ↓
Weight
 ↓
예측
 ↓
정답과 비교
 ↓
Loss
 ↓
Weight 수정
```

LLM의 학습을 매우 단순화하면,

```text
문장
 ↓
Token
 ↓
Embedding
 ↓
Transformer
 ↓
다음 Token 확률 예측
 ↓
정답 Token과 비교
 ↓
Loss
 ↓
Backpropagation
 ↓
수많은 Weight 수정
```

과 같은 형태로 볼 수 있습니다.

즉 규모와 모델 구조는 크게 다르지만 **Parameter를 이용해 예측하고 Loss를 계산한 뒤 Gradient를 이용해 Parameter를 조정한다**는 학습의 기본 틀을 이해하는 데 현재 예제가 도움이 됩니다.

---

# 19. 시뮬레이터에서 직접 확인할 것

함께 만든 HTML 프로그램에서는 다음 값을 직접 변경할 수 있습니다.

### 시작 가중치

```text
0.3
```

처음 AI가 어느 위치에서 시작하는지 결정합니다.

### Learning Rate

```text
0.01
```

한 번에 Weight를 얼마나 수정할지 결정합니다.

### Epoch

```text
300
```

얼마나 반복해서 학습할지 결정합니다.

### 1 Epoch 실행

한 번씩 눌러보면,

```text
Weight
Gradient
Loss
```

가 어떻게 변하는지 단계별로 볼 수 있습니다.

---

# 20. 그래프는 무엇을 보여주는가?

시뮬레이터의 그래프에는 정답과 현재 AI의 예측이 함께 표시됩니다.

처음에는,

```text
정답

y = 2x


예측

y = 0.3x
```

이므로 두 선의 차이가 큽니다.

학습이 진행되면서,

```text
y = 0.3x
 ↓
y = 0.8x
 ↓
y = 1.5x
 ↓
y = 1.9x
 ↓
y ≈ 2x
```

로 변합니다.

그래프에서 두 선이 점점 겹치는 것이 바로 **학습되고 있는 모습을 시각적으로 표현한 것**입니다.

---

# 21. 전체 프로그램 소스

아래 코드를 `ai_learning_simulator.html`로 저장하면 브라우저에서 바로 실행할 수 있습니다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>AI 학습 원리 시뮬레이터</title>
<style>
body {
  font-family: sans-serif;
  max-width: 900px;
  margin: 30px auto;
  padding: 0 20px;
}
.controls {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  margin: 20px 0;
}
input {
  width: 100px;
  padding: 7px;
}
button {
  padding: 8px 14px;
}
table {
  border-collapse: collapse;
  width: 100%;
  margin-top: 15px;
}
th, td {
  border-bottom: 1px solid #ddd;
  padding: 8px;
  text-align: right;
}
</style>
</head>
<body>

<h1>AI 학습 원리 시뮬레이터</h1>

<p>
정답 공식 y = 2x를 알려주지 않고,
학습 데이터를 이용해 가중치 w를 찾습니다.
</p>

<div class="controls">
  <label>
    시작 Weight
    <input id="startWeight" type="number" step="0.1" value="0.3">
  </label>

  <label>
    Learning Rate
    <input id="learningRate" type="number" step="0.001" value="0.01">
  </label>

  <label>
    Epoch
    <input id="epochs" type="number" value="300">
  </label>
</div>

<button id="trainBtn">학습 시작</button>
<button id="stepBtn">1 Epoch 실행</button>
<button id="resetBtn">초기화</button>

<h2>현재 상태</h2>

<p>Epoch: <span id="epochVal">0</span></p>
<p>Weight: <span id="weightVal">0.3</span></p>
<p>Loss: <span id="lossVal">-</span></p>

<table>
<thead>
<tr>
  <th>x</th>
  <th>정답</th>
  <th>예측</th>
</tr>
</thead>
<tbody id="predBody"></tbody>
</table>

<script>
const data = [
  {x:1, y:2},
  {x:2, y:4},
  {x:3, y:6},
  {x:4, y:8}
];

let w = 0.3;
let epoch = 0;
let training = false;

function predict(x) {
  return x * w;
}

function loss() {
  let total = 0;

  for (const d of data) {
    const e = predict(d.x) - d.y;
    total += e * e;
  }

  return total / data.length;
}

function gradient() {
  let total = 0;

  for (const d of data) {
    const e = predict(d.x) - d.y;
    total += 2 * e * d.x;
  }

  return total / data.length;
}

function trainOne() {
  const lr =
    Number(document.getElementById("learningRate").value);

  const g = gradient();

  w = w - lr * g;

  epoch++;

  update();
}

function update() {
  document.getElementById("epochVal").textContent = epoch;
  document.getElementById("weightVal").textContent =
    w.toFixed(6);
  document.getElementById("lossVal").textContent =
    loss().toFixed(8);

  const body = document.getElementById("predBody");
  body.innerHTML = "";

  for (const d of data) {
    const row = document.createElement("tr");

    row.innerHTML =
      "<td>" + d.x + "</td>" +
      "<td>" + d.y + "</td>" +
      "<td>" + predict(d.x).toFixed(4) + "</td>";

    body.appendChild(row);
  }
}

function reset() {
  training = false;

  w = Number(
    document.getElementById("startWeight").value
  );

  epoch = 0;

  update();
}

async function train() {
  if (training) return;

  training = true;

  const maxEpoch =
    Number(document.getElementById("epochs").value);

  while (training && epoch < maxEpoch) {
    trainOne();

    await new Promise(
      resolve => setTimeout(resolve, 20)
    );
  }

  training = false;
}

document
  .getElementById("trainBtn")
  .addEventListener("click", train);

document
  .getElementById("stepBtn")
  .addEventListener("click", () => {
    if (!training) trainOne();
  });

document
  .getElementById("resetBtn")
  .addEventListener("click", reset);

reset();
</script>

</body>
</html>
```

---

# 22. 이 예제에서 반드시 이해해야 할 핵심

이 프로그램을 통해 가장 먼저 이해해야 하는 것은 AI가 무언가를 신비롭게 생각해서 정답을 찾아내는 것이 아니라는 점입니다.

현재 프로그램에서는,

```text
데이터
 ↓
수학적 계산
 ↓
예측
 ↓
오차
 ↓
미분
 ↓
Parameter 수정
```

이라는 계산이 반복됩니다.

처음에는 잘못된 값을 가지고 있습니다.

```text
w = 0.3
```

하지만 데이터에 대한 예측 오차를 계산하고,

```text
Loss
```

Loss를 줄일 방향을 계산하고,

```text
Gradient
```

가중치를 조금 수정합니다.

```text
w = w - learningRate × gradient
```

이것을 반복하면,

```text
w ≈ 2
```

에 도달합니다.

따라서 이 작은 예제에서 AI 학습의 핵심을 한 문장으로 표현하면 다음과 같습니다.

> **데이터를 이용해 예측하고, 예측의 오차가 작아지는 방향으로 모델의 Parameter를 반복해서 수정하는 과정이 학습이다.**

---

# 23. 다음 단계

현재 프로그램은,

```text
입력 1개
Weight 1개
출력 1개
```

뿐입니다.

다음 단계에서는 다음 순서로 확장할 수 있습니다.

```text
1단계
y = wx

      ↓

2단계
y = wx + b

      ↓

3단계
입력 여러 개

      ↓

4단계
뉴런 여러 개

      ↓

5단계
은닉층 추가

      ↓

6단계
활성화 함수

      ↓

7단계
Backpropagation

      ↓

8단계
XOR 학습

      ↓

9단계
손글씨 숫자 분류

      ↓

10단계
Transformer와 LLM 원리
```

특히 다음 단계인 **`y = wx + b`와 뉴런 하나를 직접 구현하는 과정**까지 이해하면 단순한 선형 회귀에서 실제 신경망의 기본 구조로 자연스럽게 넘어갈 수 있습니다.
