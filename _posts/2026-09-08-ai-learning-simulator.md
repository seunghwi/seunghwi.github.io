---
layout: single
title: "AI 학습 시뮬레이터"
categories:
  - "AI"
permalink: /topics/ai/learning-simulator/
tags:
  - "AI"
  - "LLM"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>AI 학습 원리 시뮬레이터</title>
<style>
  :root {
    --bg: #f6f7fb;
    --card: #ffffff;
    --text: #1f2937;
    --muted: #6b7280;
    --line: #d1d5db;
    --accent: #111827;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    background: var(--bg);
    color: var(--text);
  }
  .wrap {
    max-width: 1180px;
    margin: 0 auto;
    padding: 28px 18px 48px;
  }
  h1 { margin: 0 0 8px; font-size: 30px; }
  .lead { color: var(--muted); margin-bottom: 24px; line-height: 1.6; }
  .grid {
    display: grid;
    grid-template-columns: 1fr 1.2fr 1fr;
    gap: 16px;
  }
  .card {
    background: var(--card);
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    padding: 18px;
    box-shadow: 0 6px 20px rgba(0,0,0,.04);
  }
  .card h2 {
    font-size: 18px;
    margin: 0 0 12px;
  }
  .muted { color: var(--muted); font-size: 13px; line-height: 1.5; }
  table { width: 100%; border-collapse: collapse; margin-top: 10px; }
  th, td { padding: 8px 10px; border-bottom: 1px solid #eee; text-align: right; }
  th:first-child, td:first-child { text-align: left; }
  .controls {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin: 18px 0;
  }
  label { display: block; font-size: 13px; color: var(--muted); margin-bottom: 6px; }
  input {
    width: 100%;
    padding: 10px 11px;
    border: 1px solid #d1d5db;
    border-radius: 9px;
    font-size: 14px;
  }
  .buttons { display: flex; gap: 10px; flex-wrap: wrap; margin-top: 12px; }
  button {
    border: 0;
    padding: 10px 14px;
    border-radius: 9px;
    cursor: pointer;
    font-weight: 700;
    background: #111827;
    color: white;
  }
  button.secondary {
    background: #e5e7eb;
    color: #111827;
  }
  .stats {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
    margin-top: 12px;
  }
  .stat {
    padding: 12px;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    background: #fafafa;
  }
  .stat b { display: block; font-size: 20px; margin-top: 4px; }
  canvas {
    width: 100%;
    height: 280px;
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
  }
  .flow {
    font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
    line-height: 1.7;
    font-size: 14px;
    white-space: pre-wrap;
    background: #f9fafb;
    border: 1px solid #eee;
    border-radius: 10px;
    padding: 12px;
  }
  .log {
    max-height: 215px;
    overflow: auto;
    font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
    font-size: 12px;
    background: #0f172a;
    color: #e5e7eb;
    border-radius: 10px;
    padding: 12px;
  }
  .explain {
    margin-top: 18px;
    line-height: 1.7;
  }
  .explain code { background: #f3f4f6; padding: 2px 5px; border-radius: 5px; }
  @media (max-width: 900px) {
    .grid { grid-template-columns: 1fr; }
    .controls { grid-template-columns: 1fr 1fr; }
  }
</style>
</head>
<body>
<div class="wrap">
  <h1>AI 학습 원리 시뮬레이터</h1>
  <div class="lead">
    정답 공식 <b>y = 2x</b>를 알려주지 않고, 데이터만 보고 가중치 <b>w</b>를 스스로 찾게 합니다.
    핵심은 <b>예측 → 오차 계산 → 가중치 수정</b>을 반복하는 것입니다.
  </div>

  <div class="grid">
    <section class="card">
      <h2>1. 학습 데이터</h2>
      <div class="muted">AI는 아래 데이터만 봅니다. y = 2x라는 공식을 직접 알려주지 않습니다.</div>
      <table>
        <thead><tr><th>x</th><th>정답 y</th></tr></thead>
        <tbody id="dataBody"></tbody>
      </table>

      <div class="flow" style="margin-top:14px;">입력 x
  ↓
예측 ŷ = x × w
  ↓
정답 y와 비교
  ↓
Loss 계산
  ↓
Gradient 계산
  ↓
w 수정</div>
    </section>

    <section class="card">
      <h2>2. 학습 과정</h2>
      <canvas id="chart" width="700" height="360"></canvas>
      <div class="stats">
        <div class="stat">현재 Epoch<b id="epochVal">0</b></div>
        <div class="stat">Weight w<b id="weightVal">0.3000</b></div>
        <div class="stat">Loss<b id="lossVal">-</b></div>
      </div>
    </section>

    <section class="card">
      <h2>3. 현재 예측</h2>
      <table>
        <thead><tr><th>x</th><th>정답</th><th>예측</th></tr></thead>
        <tbody id="predBody"></tbody>
      </table>
      <div class="log" id="log"></div>
    </section>
  </div>

  <div class="controls">
    <div>
      <label>시작 가중치 w</label>
      <input id="startWeight" type="number" step="0.1" value="0.3">
    </div>
    <div>
      <label>학습률 Learning Rate</label>
      <input id="learningRate" type="number" step="0.001" value="0.01">
    </div>
    <div>
      <label>전체 Epoch</label>
      <input id="epochs" type="number" step="10" value="300">
    </div>
    <div>
      <label>애니메이션 간격(ms)</label>
      <input id="delay" type="number" step="5" value="20">
    </div>
  </div>

  <div class="buttons">
    <button id="trainBtn">학습 시작</button>
    <button id="stepBtn" class="secondary">1 Epoch 실행</button>
    <button id="resetBtn" class="secondary">초기화</button>
  </div>

  <div class="card explain">
    <h2>무슨 일이 일어나는가?</h2>
    <p><b>모델:</b> 이 프로그램의 AI는 아주 단순하게 <code>예측 = x × w</code> 하나만 사용합니다.</p>
    <p><b>Loss:</b> 예측과 정답의 차이를 제곱해서 평균냅니다. 이것이 평균제곱오차(MSE)입니다.</p>
    <p><b>Gradient:</b> 현재 w를 어느 방향으로 얼마나 움직이면 Loss가 줄어드는지 계산합니다.</p>
    <p><b>Gradient Descent:</b> <code>w = w - 학습률 × gradient</code> 방식으로 조금씩 w를 수정합니다.</p>
    <p>학습이 잘 되면 w는 점점 <b>2</b>에 가까워집니다. 학습률을 너무 크게 바꿔서 다시 실행해보면 값이 진동하거나 발산하는 것도 확인할 수 있습니다.</p>
  </div>
</div>

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
let lossHistory = [];

const dataBody = document.getElementById('dataBody');
const predBody = document.getElementById('predBody');
const logEl = document.getElementById('log');
const canvas = document.getElementById('chart');
const ctx = canvas.getContext('2d');

data.forEach(d => {
  const tr = document.createElement('tr');
  tr.innerHTML = `<td>${d.x}</td><td>${d.y}</td>`;
  dataBody.appendChild(tr);
});

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
  const lr = Number(document.getElementById('learningRate').value);
  const g = gradient();
  const oldW = w;
  w = w - lr * g;
  epoch++;
  const l = loss();
  lossHistory.push(l);
  log(`Epoch ${epoch}: w ${oldW.toFixed(4)} → ${w.toFixed(4)}, gradient=${g.toFixed(4)}, loss=${l.toFixed(6)}`);
  update();
}

function log(msg) {
  const div = document.createElement('div');
  div.textContent = msg;
  logEl.prepend(div);
  while (logEl.children.length > 120) logEl.removeChild(logEl.lastChild);
}

function updatePredictions() {
  predBody.innerHTML = '';
  for (const d of data) {
    const tr = document.createElement('tr');
    tr.innerHTML = `<td>${d.x}</td><td>${d.y.toFixed(2)}</td><td>${predict(d.x).toFixed(4)}</td>`;
    predBody.appendChild(tr);
  }
}

function updateStats() {
  document.getElementById('epochVal').textContent = epoch;
  document.getElementById('weightVal').textContent = w.toFixed(4);
  document.getElementById('lossVal').textContent = loss().toFixed(6);
}

function drawChart() {
  const W = canvas.width, H = canvas.height;
  ctx.clearRect(0,0,W,H);
  ctx.fillStyle = '#fff';
  ctx.fillRect(0,0,W,H);

  const pad = 46;
  const xMax = 4.5;
  const yMax = 9.5;

  ctx.strokeStyle = '#d1d5db';
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(pad, H-pad);
  ctx.lineTo(W-pad, H-pad);
  ctx.moveTo(pad, H-pad);
  ctx.lineTo(pad, pad);
  ctx.stroke();

  ctx.fillStyle = '#6b7280';
  ctx.font = '13px sans-serif';
  for (let i=0; i<=4; i++) {
    const x = pad + (W-2*pad)*(i/xMax);
    ctx.fillText(String(i), x-4, H-pad+20);
  }
  for (let i=0; i<=8; i+=2) {
    const y = H-pad - (H-2*pad)*(i/yMax);
    ctx.fillText(String(i), pad-26, y+4);
  }

  function px(x){ return pad + (W-2*pad)*(x/xMax); }
  function py(y){ return H-pad - (H-2*pad)*(y/yMax); }

  // 정답선 y=2x
  ctx.strokeStyle = '#111827';
  ctx.lineWidth = 2;
  ctx.setLineDash([8,6]);
  ctx.beginPath();
  ctx.moveTo(px(0), py(0));
  ctx.lineTo(px(4.5), py(9));
  ctx.stroke();
  ctx.setLineDash([]);

  // 예측선 y=w*x
  ctx.strokeStyle = '#6b7280';
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.moveTo(px(0), py(0));
  ctx.lineTo(px(4.5), py(4.5*w));
  ctx.stroke();

  // 데이터 점
  ctx.fillStyle = '#111827';
  for (const d of data) {
    ctx.beginPath();
    ctx.arc(px(d.x), py(d.y), 6, 0, Math.PI*2);
    ctx.fill();
  }

  ctx.fillStyle = '#111827';
  ctx.fillText('점선: 정답 y=2x', W-180, 25);
  ctx.fillStyle = '#6b7280';
  ctx.fillText(`실선: 예측 y=${w.toFixed(3)}x`, W-180, 45);
}

function update() {
  updatePredictions();
  updateStats();
  drawChart();
}

function reset() {
  training = false;
  w = Number(document.getElementById('startWeight').value);
  epoch = 0;
  lossHistory = [];
  logEl.innerHTML = '';
  log(`초기화: w=${w}`);
  update();
}

async function train() {
  if (training) return;
  training = true;
  const maxEpochs = Number(document.getElementById('epochs').value);
  const delay = Number(document.getElementById('delay').value);

  while (training && epoch < maxEpochs) {
    trainOne();

    if (!Number.isFinite(w) || Math.abs(w) > 1e12) {
      log('학습이 발산했습니다. 학습률을 낮춰보세요.');
      training = false;
      break;
    }

    if (loss() < 1e-10) {
      log('거의 완벽하게 학습했습니다.');
      training = false;
      break;
    }

    await new Promise(r => setTimeout(r, delay));
  }
  training = false;
}

document.getElementById('trainBtn').addEventListener('click', train);
document.getElementById('stepBtn').addEventListener('click', () => {
  if (!training) trainOne();
});
document.getElementById('resetBtn').addEventListener('click', reset);
document.getElementById('startWeight').addEventListener('change', reset);

reset();
</script>
</body>
</html>
