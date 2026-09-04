---
layout: single
title: "AI 서빙 프레임워크 완전 가이드: Ollama·vLLM·SGLang 비교와 선택"
categories:
  - "AI"
permalink: /topics/ai/llm-serving-frameworks/
tags:
  - "AI"
  - "LLM"
  - "AI 서빙"
  - "Ollama"
  - "vLLM"
  - "SGLang"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

학습을 마친 LLM 파일이 있다고 해서 곧바로 여러 사용자가 안정적으로 이용할 수 있는 서비스가 되는 것은 아닙니다. 모델을 가속기 메모리에 올리고, 동시에 들어오는 요청을 묶어 계산하고, 각 요청의 KV 캐시를 관리하며, 생성되는 토큰을 실시간으로 전송해야 합니다. 과부하와 장애를 통제하고 사용량도 관측해야 합니다.

이 과정을 담당하는 것이 **AI 서빙 프레임워크**, 그중에서도 **LLM 추론·서빙 엔진**입니다. 이 글은 서빙의 내부 동작부터 Ollama, vLLM, SGLang의 구조와 장단점, 설치·호출 예제, 운영 지표와 선택 기준까지 단계적으로 설명합니다. llama.cpp, TensorRT-LLM, Hugging Face TGI, Triton 같은 인접 선택지도 함께 살펴봅니다.

이 글은 **2026년 9월의 공식 문서**를 기준으로 작성했습니다. 서빙 프레임워크는 지원 모델·가속기·양자화 방식과 명령행 옵션이 빠르게 바뀝니다. 실제 도입 전에는 각 프로젝트의 지원 목록과 최신 릴리스 문서를 다시 확인해야 합니다.
{: .notice--warning}

## 먼저 결론: 세 프레임워크의 역할은 조금씩 다르다

| 프레임워크 | 중심 목표 | 가장 잘 맞는 환경 | 한 문장 요약 |
|---|---|---|---|
| **Ollama** | 로컬에서 모델을 쉽게 내려받고 실행 | 개인 PC, 개발, 데모, 사내 단일 사용자 도구 | 모델 관리와 로컬 API를 한 번에 제공하는 사용성 중심 도구 |
| **vLLM** | 범용적인 고성능 LLM 서빙 | GPU 서버, 다중 사용자 API, 연구·프로덕션 | PagedAttention과 연속 배치가 강점인 범용 서빙 엔진 |
| **SGLang** | 복잡한 생성 작업과 대규모 고성능 서빙 | 공유 프롬프트, 에이전트, 구조화 출력, 분산 GPU | RadixAttention과 정교한 런타임 최적화가 강점인 서빙 프레임워크 |

따라서 “무조건 가장 빠른 프레임워크”를 고르는 문제로 접근하면 안 됩니다. 노트북에서 모델을 시험하는 사람에게는 Ollama의 간결함이 중요하고, 많은 사용자의 요청을 GPU 서버로 처리한다면 vLLM이나 SGLang의 스케줄러와 분산 기능이 중요합니다.

같은 모델도 입력·출력 길이, 동시 요청 수, 반복되는 접두사, 양자화, GPU 세대와 서비스 목표에 따라 결과가 바뀝니다. 벤치마크 표 한 장만 보고 선택하지 말고 실제 트래픽을 재현해야 합니다.
{: .notice--info}

## AI 서빙 프레임워크란 무엇인가

### 학습, 추론, 서빙의 차이

- **학습(training)**은 데이터로 모델 파라미터를 변경하는 과정입니다.
- **추론(inference)**은 고정된 파라미터로 입력에 대한 출력을 계산하는 과정입니다.
- **서빙(serving)**은 추론을 네트워크 API로 제공하면서 동시성, 지연 시간, 자원, 장애와 보안을 관리하는 과정입니다.

`transformers`의 `model.generate()`를 한 번 실행하는 것도 추론입니다. 그러나 여러 클라이언트가 동시에 접속할 때 요청을 효율적으로 합치고, 취소된 요청의 메모리를 회수하고, 토큰을 스트리밍하며, 상태와 지표를 노출하는 것은 서빙의 영역입니다.

### 애플리케이션 프레임워크와도 다르다

LangChain이나 LangGraph는 프롬프트, 검색, 도구 호출과 실행 흐름을 조립하는 **애플리케이션 계층**입니다. Ollama, vLLM과 SGLang Runtime은 모델의 순전파와 토큰 생성을 효율화하는 **추론 계층**에 가깝습니다. 둘은 경쟁 제품이 아니라 함께 사용할 수 있습니다.

~~~text
사용자 애플리케이션
  └─ LangChain·LangGraph·직접 작성한 백엔드
       └─ API 게이트웨이·인증·요청 제한·라우팅
            └─ Ollama / vLLM / SGLang
                 └─ PyTorch·CUDA·ROCm·Metal·가속 커널
                      └─ GPU·CPU·NPU
~~~

KServe, Ray Serve와 Kubernetes는 복제본 배치, 오토스케일링과 트래픽 라우팅을 담당할 수 있습니다. 이들 역시 GPU 안에서 KV 캐시와 토큰 생성을 직접 최적화하는 엔진과는 역할이 다릅니다.

## 한 요청이 응답으로 바뀌는 과정

~~~text
HTTP 요청
  → 인증·대기열
  → 채팅 템플릿 적용
  → 토큰화
  → 스케줄러와 배치 구성
  → Prefill: 입력 전체 처리
  → KV 캐시 저장
  → Decode: 다음 토큰을 하나씩 반복 생성
  → 디토큰화·스트리밍
  → 사용량·지연 시간 기록
~~~

### 1. 채팅 템플릿과 토큰화

`system`, `user`, `assistant` 메시지는 모델이 이해하는 특수 토큰 문자열로 변환됩니다. 모델마다 템플릿이 다르므로 잘못된 템플릿을 쓰면 서버는 정상 응답해도 품질과 도구 호출 형식이 무너질 수 있습니다. 이어서 텍스트가 토큰 ID 배열로 바뀝니다.

### 2. Prefill

Prefill은 입력 토큰 전체를 병렬로 처리해 첫 번째 출력 토큰을 준비합니다. 긴 문서나 대화 기록을 넣을수록 계산량이 커집니다. 대체로 GPU 연산 능력에 민감한 **compute-bound** 단계이며, 사용자가 느끼는 **첫 토큰까지의 시간(TTFT)**에 큰 영향을 줍니다.

### 3. KV 캐시

Transformer는 새 토큰을 생성할 때 이전 모든 토큰의 attention 정보를 다시 계산할 필요가 없도록 각 층의 Key와 Value를 저장합니다. 이것이 **KV 캐시**입니다. 계산을 크게 줄여 주지만 요청 수와 문맥 길이에 비례해 가속기 메모리를 차지합니다.

모델 가중치가 GPU에 들어간다는 사실만으로 충분하지 않습니다. 남은 메모리에 활성값, 작업 공간과 동시 요청의 KV 캐시가 들어가야 실제 처리량이 나옵니다.
{: .notice--info}

### 4. Decode

Decode는 직전 결과를 바탕으로 다음 토큰 하나를 만들고 이를 반복합니다. 한 단계의 계산량보다 모델 가중치와 KV 캐시를 계속 읽는 비용이 커서 대체로 메모리 대역폭에 민감합니다. 이때 사용자가 느끼는 생성 속도는 **토큰 간 지연(ITL)** 또는 **첫 토큰 이후 토큰당 시간(TPOT)**으로 측정합니다.

## 일반 웹 서버만으로 부족한 이유

일반적인 이미지 분류는 입력 크기와 한 번의 순전파 시간이 비교적 일정합니다. 반면 LLM 요청은 다음 특성이 있습니다.

- 프롬프트와 출력 길이가 요청마다 크게 다릅니다.
- 한 요청이 수백 번의 decode 반복을 수행할 수 있습니다.
- 실행 도중 다른 요청이 도착하거나 완료되고 취소됩니다.
- KV 캐시가 가변적으로 커져 메모리 단편화와 부족을 일으킵니다.
- 스트리밍을 위해 완성 전부터 토큰을 돌려줘야 합니다.
- 긴 prefill 하나가 짧은 decode 요청을 방해할 수 있습니다.

그래서 LLM 서버는 단순한 HTTP 래퍼가 아니라 **반복 단위 스케줄러와 메모리 관리자**여야 합니다.

## 성능을 만드는 핵심 기술

### 정적 배치와 연속 배치

정적 배치는 여러 요청을 한 묶음으로 시작하고 모두 끝날 때까지 기다립니다. 짧은 요청이 끝나도 가장 긴 요청 때문에 슬롯이 비어 있을 수 있습니다.

**연속 배치(continuous batching, in-flight batching)**는 생성 단계마다 실행 중인 요청을 다시 구성합니다. 끝난 요청을 제거하고 대기 중인 새 요청을 즉시 넣어 GPU가 쉬는 시간을 줄입니다.

~~~text
정적 배치:  [A A A 끝][B B B B B B 끝] → 묶음 전체 종료 후 새 요청
연속 배치:  [A A A 끝]
            [B B B B B B 끝]
                  [C C C C 끝]          → 빈 자리에 C 투입
~~~

처리량은 늘지만 배치를 너무 공격적으로 키우면 요청별 지연이 악화될 수 있습니다. 좋은 설정은 GPU 사용률 자체보다 정해진 SLO 안에서 완료한 요청의 양, 즉 **goodput**을 높입니다.

### PagedAttention과 페이지형 KV 캐시

연속된 큰 메모리 공간을 요청마다 미리 잡으면 실제 출력 길이가 예상보다 짧을 때 낭비가 생기고, 요청이 들고날 때 단편화됩니다. vLLM의 [PagedAttention 논문](https://arxiv.org/abs/2309.06180)은 운영체제의 가상 메모리처럼 KV 캐시를 고정 크기 블록으로 나누고 논리 블록을 비연속 물리 블록에 매핑합니다.

필요한 만큼 블록을 배정하고 완료 시 돌려줄 수 있어 더 많은 요청을 같은 메모리에 담을 수 있습니다. vLLM에서 널리 알려졌지만 오늘날 여러 엔진이 페이지형 KV 캐시 개념을 사용합니다.

### Prefix caching과 RadixAttention

긴 시스템 프롬프트, few-shot 예시 또는 이전 대화가 여러 요청에서 같다면 그 접두사의 KV 캐시를 재사용할 수 있습니다. 일반적인 prefix caching은 동일한 토큰 블록을 찾아 중복 prefill을 줄입니다.

SGLang의 **RadixAttention**은 토큰 접두사와 KV 캐시를 radix tree로 연결해 여러 요청·단계 사이에서 자동으로 재사용하고 제거합니다. 반복되는 긴 시스템 프롬프트, 멀티턴 대화, 분기형 에이전트와 RAG 워크로드에서 특히 유리할 수 있습니다. 반대로 매 요청의 시작부터 모두 다르면 캐시 적중 이점은 작습니다. 자세한 설계는 [SGLang 논문](https://arxiv.org/abs/2312.07104)에 설명되어 있습니다.

### Chunked prefill

매우 긴 입력 하나를 한 번에 prefill하면 GPU를 오래 점유해 다른 요청의 토큰 스트리밍이 끊길 수 있습니다. **chunked prefill**은 긴 입력을 작은 조각으로 나누어 decode 작업과 섞어 실행합니다. 처리량과 메모리 사용을 조절하고 tail latency를 줄일 수 있지만 청크와 스케줄러 설정에 따라 오버헤드가 생깁니다.

### Speculative decoding

작은 draft 모델이나 예측 헤드가 여러 후보 토큰을 먼저 만들고 큰 target 모델이 한 번에 검증합니다. 후보가 자주 승인되면 큰 모델의 순전파 횟수가 줄어 지연 시간이 개선됩니다. 다만 추가 모델 메모리, 검증 비용과 설정 복잡도가 있고, 승인률이 낮은 작업에서는 이득이 작거나 손해가 날 수 있습니다.

### 양자화

가중치를 FP16/BF16보다 작은 FP8, INT8, INT4 등으로 표현하면 메모리 사용량과 대역폭을 줄일 수 있습니다. 더 큰 모델을 한 GPU에 넣거나 동시 요청용 KV 캐시를 늘리는 데 유리합니다.

그러나 “4비트면 항상 두 배 빠르다”는 식으로 계산할 수는 없습니다. 하드웨어가 해당 형식을 얼마나 잘 지원하는지, dequantization 커널, 모델 구조와 batch 크기에 따라 성능이 달라집니다. 정확도와 도구 호출 안정성도 대표 데이터로 다시 평가해야 합니다.

### 병렬화

| 방식 | 무엇을 나누는가 | 장점 | 주요 비용 |
|---|---|---|---|
| Tensor Parallelism, TP | 한 층의 행렬 | 한 모델을 여러 GPU에 분산 | 매 층 통신, 빠른 인터커넥트 필요 |
| Pipeline Parallelism, PP | 모델의 층 | 노드·GPU에 층을 분배 | 파이프라인 버블과 스케줄 복잡도 |
| Data Parallelism, DP | 모델 복제본과 요청 | 총 처리량·가용성 확대 | 복제본마다 가중치 메모리 필요 |
| Expert Parallelism, EP | MoE의 전문가 | 대형 MoE 모델 분산 | 토큰 라우팅과 all-to-all 통신 |

모델이 단일 GPU에 들어가면 TP를 무조건 늘리는 것이 이득은 아닙니다. 통신 오버헤드가 생기므로 단일 요청 지연과 전체 처리량을 모두 측정해야 합니다.

### Prefill·Decode 분리

Prefill과 decode는 자원 특성이 다릅니다. 대규모 환경에서는 두 단계를 별도 워커 풀로 분리하고 KV 캐시를 네트워크로 전달할 수 있습니다. 긴 프롬프트가 토큰 생성을 방해하는 현상을 줄이고 단계별로 독립 확장할 수 있지만, 고속 네트워크와 분산 캐시 관리가 필요해 운영 복잡도가 크게 증가합니다.

## Ollama: 로컬 모델 사용 경험을 단순하게

[Ollama](https://github.com/ollama/ollama)는 모델 다운로드, 저장, 설정과 실행, 대화형 CLI와 HTTP API를 하나의 경험으로 묶습니다. macOS, Windows와 Linux에서 사용할 수 있고 내부 추론 기반에는 llama.cpp가 포함됩니다.

### 기본 사용

[공식 Quickstart](https://docs.ollama.com/quickstart)에 따라 설치한 뒤 모델을 실행합니다.

~~~bash
ollama run gemma3
~~~

첫 실행에서는 모델을 내려받고, 이후에는 로컬 캐시에서 불러옵니다. 별도 애플리케이션에서는 기본 주소 `http://localhost:11434/api`로 요청합니다.

~~~bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3",
  "messages": [
    {"role": "user", "content": "KV 캐시를 한 문장으로 설명해줘."}
  ],
  "stream": false
}'
~~~

`stream`을 생략하거나 `true`로 설정하면 생성되는 응답 조각이 순차적으로 전달됩니다. 모델 이름과 사용 가능 여부는 [Ollama 모델 라이브러리](https://ollama.com/library)에서 확인합니다.

### Modelfile로 실행 설정 묶기

Ollama의 [Modelfile](https://docs.ollama.com/modelfile)은 기반 모델, 시스템 메시지와 추론 파라미터를 선언합니다.

~~~dockerfile
FROM gemma3

PARAMETER temperature 0.2
PARAMETER num_ctx 8192

SYSTEM """
당신은 근거와 한계를 함께 설명하는 한국어 기술 지원 도우미입니다.
"""
~~~

~~~bash
ollama create technical-helper -f Modelfile
ollama run technical-helper
~~~

Safetensors 모델, GGUF 모델과 일부 adapter를 가져올 수도 있지만 구조와 양자화 지원 범위가 다릅니다. 임의 파일을 변환하기 전에 [Importing a Model 공식 문서](https://docs.ollama.com/import)를 확인해야 합니다.

### Ollama의 장점

- 설치 후 모델 이름만으로 다운로드와 실행이 가능합니다.
- 로컬 실행, CLI, REST API와 간단한 모델 설정이 한 제품에 묶여 있습니다.
- GGUF 양자화 모델을 사용해 소비자용 하드웨어에서도 비교적 쉽게 시작할 수 있습니다.
- Apple Silicon을 포함한 개인용 환경에서 프로토타이핑하기 좋습니다.
- 네트워크로 외부 제공자에게 프롬프트를 보내지 않는 완전 로컬 구성이 가능합니다.
- [OpenAI 호환 API](https://docs.ollama.com/api/openai-compatibility)를 제공해 기존 클라이언트를 연결하기 쉽습니다.

### Ollama의 단점과 주의점

- 핵심 목표가 대규모 다중 GPU 클러스터의 최대 처리량은 아닙니다.
- vLLM·SGLang처럼 분산 병렬화와 세부 스케줄링을 폭넓게 조정하려는 운영에는 선택지가 제한적입니다.
- 동일 모델 이름이라도 양자화와 템플릿이 다른 서버와 달라 품질·성능 비교가 왜곡될 수 있습니다.
- 로컬 실행이 자동으로 안전을 의미하지는 않습니다. 디스크의 모델과 대화 데이터 접근 권한을 관리해야 합니다.
- 로컬 API는 기본적으로 `localhost`에서 인증을 요구하지 않습니다. 외부 인터페이스에 그대로 노출하지 말고 인증·TLS·요청 제한을 적용한 프록시 뒤에 둬야 합니다. 자세한 내용은 [Ollama 인증 문서](https://docs.ollama.com/api/authentication)를 참고합니다.

### Ollama가 잘 맞는 경우

- 노트북에서 여러 공개 모델을 빠르게 비교할 때
- 인터넷 연결 없이 개인 문서 요약이나 개발 보조를 실행할 때
- 교육, 데모와 소규모 사내 도구의 첫 버전을 만들 때
- LangChain 같은 애플리케이션 프레임워크의 로컬 백엔드가 필요할 때

## vLLM: 범용 고성능 LLM 서버

[vLLM](https://docs.vllm.ai/en/latest/)은 높은 처리량과 효율적인 메모리 사용을 목표로 하는 오픈 소스 LLM 추론·서빙 엔진입니다. PagedAttention을 중심으로 연속 배치, prefix caching, chunked prefill, speculative decoding과 여러 병렬화 방식을 제공합니다.

### 서버 실행과 호출

공식 문서의 예시 모델로 서버를 시작하면 기본적으로 `localhost:8000`에서 OpenAI 호환 API가 열립니다.

~~~bash
vllm serve Qwen/Qwen2.5-1.5B-Instruct \
  --api-key local-token
~~~

~~~python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="local-token",
)

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-1.5B-Instruct",
    messages=[
        {"role": "user", "content": "연속 배치의 장점을 설명해줘."}
    ],
    temperature=0.2,
)

print(response.choices[0].message.content)
~~~

실행 명령, 모델 ID와 옵션은 버전에 따라 달라질 수 있으므로 [vLLM Quickstart](https://docs.vllm.ai/en/latest/getting_started/quickstart/)와 [서버 인자 문서](https://docs.vllm.ai/en/latest/configuration/serve_args/)를 기준으로 적용합니다.

### vLLM의 주요 기능

- PagedAttention 기반의 효율적인 KV 캐시 관리
- 연속 배치와 chunked prefill
- 자동 prefix caching
- tensor, pipeline, data, expert parallelism
- 다양한 가중치·KV 캐시 양자화 방식
- speculative decoding
- 구조화 출력과 tool calling·reasoning parser
- 여러 LoRA adapter를 함께 제공하는 multi-LoRA
- OpenAI 호환 API와 스트리밍
- NVIDIA CUDA, AMD ROCm, CPU 및 확장 plugin을 통한 여러 하드웨어 지원

지원 여부는 모델 구조, 장치와 정밀도 조합마다 다릅니다. 문서에 기능 이름이 있다고 해서 모든 조합에서 동시에 사용할 수 있다는 의미는 아닙니다.
{: .notice--warning}

### vLLM의 장점

- 동시 요청이 있는 환경에서 메모리 효율과 처리량이 좋습니다.
- Hugging Face 모델 생태계와 OpenAI 방식 API에 연결하기 쉽습니다.
- 단일 GPU에서 다중 GPU·노드로 확장하는 기능 범위가 넓습니다.
- 커뮤니티와 통합 생태계가 크고 다양한 모델 아키텍처를 지원합니다.
- 오프라인 일괄 추론과 온라인 API 서버를 모두 지원합니다.
- 성능 관련 옵션이 풍부해 실제 트래픽에 맞춰 조정할 수 있습니다.

### vLLM의 단점과 주의점

- Ollama보다 설치와 CUDA·ROCm 버전 호환성 관리가 복잡합니다.
- 많은 옵션이 서로 영향을 주므로 기본 실행이 곧 최적 설정은 아닙니다.
- 새 모델 구조나 특수한 양자화는 `transformers`에서 실행되더라도 vLLM 지원이 늦을 수 있습니다.
- 서버 인스턴스 하나가 모든 모델을 동적으로 교체하는 범용 모델 저장소는 아닙니다. 일반적으로 모델별 프로세스와 상위 라우터를 구성합니다.
- OpenAI 호환은 편리한 이식 계층이지 모든 제공자 확장 필드와 동작이 완전히 같다는 보장은 아닙니다.
- 고동시성 설정은 평균 처리량을 늘리면서 p99 지연과 단일 사용자의 생성 속도를 악화시킬 수 있습니다.

### vLLM이 잘 맞는 경우

- 공개 가중치 모델을 사내 OpenAI 호환 API로 제공할 때
- 단일 또는 다중 GPU에서 다수 사용자의 요청을 처리할 때
- 범용 챗봇, 요약, 코드 생성과 배치 추론을 한 엔진으로 운영할 때
- 폭넓은 모델 지원과 검증된 생태계를 우선할 때

## SGLang: 접두사 재사용과 복잡한 생성 흐름까지

[SGLang](https://docs.sglang.ai/)은 LLM과 멀티모달 모델을 위한 고성능 서빙 프레임워크입니다. 프로젝트는 모델 서버인 **SGLang Runtime(SRT)**과 구조화된 언어 모델 프로그램을 표현하는 프런트엔드에서 출발했습니다. 현재 프로덕션 서빙에서는 SRT의 고성능 런타임 기능이 중심입니다.

### 서버 실행과 OpenAI 호환 API

~~~bash
python -m sglang.launch_server \
  --model-path Qwen/Qwen2.5-1.5B-Instruct \
  --host 127.0.0.1 \
  --port 30000
~~~

~~~bash
curl http://127.0.0.1:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "messages": [
      {"role": "user", "content": "RadixAttention을 간단히 설명해줘."}
    ],
    "temperature": 0.2
  }'
~~~

설치 방식과 가속기별 패키지가 자주 바뀌므로 명령을 적용하기 전에 [SGLang 공식 설치 문서](https://docs.sglang.ai/get_started/install.html)를 확인합니다.

### RadixAttention의 의미

다음 세 요청이 있다고 가정합니다.

~~~text
요청 A: [긴 시스템 규칙] + [제품 설명] + 환불 조건은?
요청 B: [긴 시스템 규칙] + [제품 설명] + 배송 기간은?
요청 C: [긴 시스템 규칙] + [다른 문서] + 핵심은?
~~~

A와 B는 긴 앞부분 전체를 공유하고 C는 시스템 규칙까지만 공유합니다. RadixAttention은 이런 접두사 관계를 radix tree에 넣고 KV 캐시를 연결합니다. 새 요청은 가장 긴 일치 접두사의 계산 결과를 재사용합니다. 캐시 메모리가 부족하면 재사용 가능성과 최근 사용 상태에 따라 항목을 제거합니다.

이 방식은 수동으로 캐시 키를 붙이지 않아도 다양한 길이의 접두사를 찾을 수 있다는 점이 중요합니다. 다만 여러 복제본으로 확장하면 같은 접두사가 이미 있는 복제본으로 요청을 보내는 **prefix-aware routing**도 필요합니다. 단순 round-robin은 각 서버의 캐시 적중률을 떨어뜨릴 수 있습니다.

### SGLang의 주요 기능

- RadixAttention 기반 자동 prefix caching
- CPU 스케줄러 최적화와 연속 배치
- 페이지형 attention과 chunked prefill
- prefill·decode 분리
- speculative decoding
- tensor, pipeline, data, expert parallelism
- FP4, FP8, INT4, AWQ, GPTQ 등 여러 양자화 경로
- JSON schema와 정규식 등을 위한 구조화 출력
- 여러 LoRA를 배치하는 multi-LoRA
- 언어 모델뿐 아니라 여러 멀티모달·embedding·reward 모델 지원
- OpenAI·Hugging Face 생태계 호환

최신 공식 소개와 구체적인 지원 목록은 [SGLang 저장소](https://github.com/sgl-project/sglang)에서 확인할 수 있습니다.

### SGLang의 장점

- 반복되는 긴 접두사가 많은 작업에서 prefill 중복을 줄이도록 설계됐습니다.
- 구조화 출력, 멀티턴, 분기와 여러 생성 호출이 있는 워크로드에 강한 최적화를 제공합니다.
- 단일 GPU부터 대형 분산·MoE 구성까지 기능 범위가 넓습니다.
- 최신 모델과 가속기 최적화를 빠르게 반영하는 활발한 프로젝트입니다.
- 자체 벤치마크 도구가 TTFT, ITL, TPOT과 처리량을 함께 측정합니다.

### SGLang의 단점과 주의점

- 설치, 커널, 통신 라이브러리와 분산 옵션의 조합이 복잡할 수 있습니다.
- 빠른 개발 속도 때문에 버전 사이의 명령, 기본값과 호환성을 세심하게 관리해야 합니다.
- RadixAttention의 이점은 접두사가 실제로 반복될 때 커집니다. 무작위·독립 프롬프트에서는 기대보다 작을 수 있습니다.
- prefill·decode 분리 같은 고급 기능은 네트워크와 KV 전송 비용까지 포함해 설계해야 합니다.
- 모든 모델·하드웨어·양자화 조합이 같은 수준으로 성숙한 것은 아닙니다.
- 작은 로컬 실험만 필요하다면 Ollama보다 학습 비용과 운영 부담이 큽니다.

### SGLang이 잘 맞는 경우

- 긴 공통 시스템 프롬프트나 few-shot 예시를 반복하는 서비스
- RAG, 멀티턴 대화와 에이전트처럼 접두사 재사용 기회가 많은 작업
- JSON schema 같은 구조화 출력을 높은 처리량으로 생성할 때
- 최신 대형 모델, MoE 또는 분산 서빙을 적극적으로 최적화할 때

## Ollama·vLLM·SGLang 상세 비교

| 비교 항목 | Ollama | vLLM | SGLang |
|---|---|---|---|
| 주 사용자 | 개인 개발자, 앱 개발자 | ML 엔지니어, 플랫폼 팀 | ML 시스템·플랫폼 팀 |
| 시작 난이도 | 낮음 | 중간 | 중간~높음 |
| 핵심 강점 | 모델 관리와 로컬 UX | 범용 고처리량과 PagedAttention | RadixAttention과 고급 런타임 |
| 대표 모델 형식 | Ollama 모델·GGUF, 가져오기 지원 | Hugging Face 계열 가중치와 여러 양자화 | Hugging Face 계열 가중치와 여러 양자화 |
| 로컬 CPU·개인 PC | 매우 적합 | 주력 환경은 서버 가속기 | 주력 환경은 서버 가속기 |
| Apple Silicon | 편리함 | 별도 생태계·plugin 상태 확인 필요 | 주력 선택지 아님 |
| 연속 배치 | 내부 런타임에 따라 처리 | 핵심 기능 | 핵심 기능 |
| Prefix caching | 사용 환경·백엔드에 따라 제한 | 자동 prefix caching | RadixAttention이 핵심 |
| 다중 GPU·분산 | 제한적 | 폭넓게 지원 | 폭넓게 지원 |
| 구조화 출력·도구 호출 | API 호환 범위 내 지원 | 지원, 모델별 parser 확인 | 강한 지원, 모델별 확인 필요 |
| 운영 튜닝 범위 | 비교적 단순 | 매우 넓음 | 매우 넓음 |
| OpenAI 호환 API | 제공 | 제공 | 제공 |
| 추천 시작점 | 로컬 PoC | 범용 프로덕션 후보 | 접두사·복합 생성 중심 후보 |

표의 “지원”은 모든 모델에서 동일한 기능을 보장한다는 뜻이 아닙니다. tool calling은 모델의 학습 방식, chat template와 parser가 함께 맞아야 하고, 양자화는 장치별 커널 지원이 맞아야 합니다.

## 다른 선택지도 알아두자

### llama.cpp

[llama.cpp](https://github.com/ggml-org/llama.cpp)는 최소한의 설정으로 다양한 하드웨어에서 LLM·VLM을 실행하는 C/C++ 프로젝트입니다. GGUF 형식을 사용하고 CPU, CUDA, HIP, Metal, Vulkan 등 폭넓은 백엔드를 지원합니다. `llama-server`로 OpenAI 호환 API도 열 수 있습니다.

- **장점:** 높은 이식성, CPU·엣지·Apple Silicon, 풍부한 GGUF 양자화, 적은 의존성
- **단점:** 대규모 다중 사용자 GPU 클러스터의 운영 기능은 vLLM·SGLang과 초점이 다름
- **추천:** 데스크톱, 임베디드·엣지, C/C++ 통합, 세밀한 로컬 제어

Ollama는 llama.cpp 기반 기능을 사용하면서 모델 설치와 관리 경험을 더 높은 수준으로 묶은 선택지라고 이해하면 쉽습니다. 직접 llama.cpp를 쓰면 제어권이 커지고, Ollama를 쓰면 시작과 배포가 간단해집니다.

### NVIDIA TensorRT-LLM

[TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/)은 NVIDIA GPU에서 LLM 추론을 최적화하는 툴킷입니다. 커널 융합, 양자화, paged KV cache, in-flight batching, speculative decoding, 다중 GPU·노드 기능을 제공합니다. 최신 계열에는 TensorRT뿐 아니라 PyTorch 백엔드와 `trtllm-serve`도 있습니다.

- **장점:** NVIDIA 하드웨어에 대한 깊은 최적화, 최신 정밀도·GPU 기능, NVIDIA 배포 생태계
- **단점:** NVIDIA 중심이며 버전·지원 행렬과 튜닝의 복잡도가 높음
- **추천:** NVIDIA 환경에서 성능을 끝까지 최적화하고 지원 조합을 통제할 수 있는 팀

### Hugging Face Text Generation Inference

[TGI](https://huggingface.co/docs/text-generation-inference/en/index)는 연속 배치, tensor parallelism, streaming, PagedAttention, 양자화, Prometheus와 OpenTelemetry를 제공해 공개 모델 서빙을 대중화한 프로젝트입니다.

다만 공식 문서는 현재 TGI가 **maintenance mode**이며, 경량 유지보수 위주로 진행된다고 밝힙니다. 기존 배포를 즉시 없애야 한다는 의미는 아니지만 새 프로젝트라면 Hugging Face가 함께 언급하는 vLLM, SGLang, llama.cpp와 현재의 `transformers serve`를 비교하는 편이 좋습니다.

### NVIDIA Triton Inference Server

[Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)는 LLM 전용 엔진이라기보다 여러 프레임워크와 모델을 함께 제공하는 범용 서버입니다. HTTP/gRPC, 모델 저장소, 동적·시퀀스 배치, 지표, 헬스 체크와 ensemble을 제공합니다.

전처리 → embedding → reranker → 생성처럼 여러 모델 파이프라인을 한 플랫폼에서 관리하거나 TensorRT-LLM backend를 운영할 때 유용합니다. 일반 동적 배치와 LLM의 토큰 단계별 연속 배치는 같은 개념이 아니므로 실제 LLM backend가 제공하는 스케줄러도 함께 봐야 합니다.

### KServe, Ray Serve와 Kubernetes

이들은 엔진 위에서 복제본, 라우팅, 배포와 오토스케일링을 관리하는 계층입니다. vLLM이나 SGLang을 컨테이너로 실행하고 상위 플랫폼이 여러 복제본에 트래픽을 분배할 수 있습니다.

LLM에서는 단순히 가장 한가한 복제본을 찾는 것만으로 충분하지 않을 수 있습니다. 기존 KV 캐시의 접두사, 모델 종류, adapter, 남은 메모리와 요청 길이를 함께 고려하는 라우팅이 성능에 중요합니다.

## API가 같아도 결과가 같지는 않다

세 프레임워크가 OpenAI 호환 API를 제공하므로 `base_url`만 바꿔 옮길 수 있는 경우가 많습니다. 그러나 다음 항목은 별도로 검증해야 합니다.

- 지원 endpoint와 request field
- `response_format`, tool calling과 reasoning field
- model ID와 alias 처리
- chat template와 기본 system prompt
- stop token, EOS와 최대 출력 길이
- `temperature`, `top_p`, seed의 의미
- streaming chunk와 오류 응답 형식
- usage의 prompt·completion token 계산
- 취소, timeout과 backpressure 동작

호환 API는 **마이그레이션 비용을 줄이는 인터페이스**이지 의미와 결과의 완전한 동일성을 보장하는 표준 인증은 아닙니다.

## 올바른 벤치마크 방법

### 먼저 서비스 목표를 정한다

| 지표 | 의미 | 사용자가 체감하는 것 |
|---|---|---|
| TTFT | 요청부터 첫 토큰까지 | 답변이 시작되는 속도 |
| ITL | 연속 토큰 사이의 시간 | 스트리밍이 매끄러운 정도 |
| TPOT | 첫 토큰 이후 토큰당 평균 시간 | 생성 속도 |
| E2E latency | 요청부터 마지막 토큰까지 | 전체 대기 시간 |
| Request throughput | 초당 완료 요청 수 | 서비스 수용량 |
| Token throughput | 초당 입력·출력 토큰 수 | 엔진 계산 처리량 |
| p95·p99 latency | 느린 요청의 지연 | 혼잡 시 품질 |
| Goodput | SLO 안에 완료된 처리량 | 실제 유효 수용량 |
| Error·reject rate | 실패·거절 비율 | 안정성과 과부하 상태 |

평균만 보면 긴 꼬리 지연을 숨깁니다. 대화형 서비스는 p95/p99 TTFT와 ITL을, 배치 작업은 총 token throughput과 비용을 더 중요하게 볼 수 있습니다.

### 공정하게 비교하는 체크리스트

1. 정확히 같은 모델 revision과 tokenizer를 고정합니다.
2. 같은 정밀도·양자화와 chat template를 사용합니다.
3. 입력·출력 길이의 분포를 실제 로그와 가깝게 만듭니다.
4. 동시 요청 수뿐 아니라 도착 간격을 재현합니다.
5. 워밍업 뒤 여러 번 측정하고 p50·p95·p99를 기록합니다.
6. prefix cache가 따뜻한 경우와 차가운 경우를 분리합니다.
7. GPU 수, 전력, 메모리 사용량과 비용을 함께 기록합니다.
8. OOM, timeout, 취소와 overload 상황도 시험합니다.
9. 속도뿐 아니라 출력 품질과 구조화 출력 성공률을 확인합니다.
10. 동일한 SLO를 만족하는 최대 부하를 찾아 비교합니다.

SGLang은 [공식 bench_serving 도구](https://docs.sglang.ai/developer_guide/bench_serving)를 통해 TTFT, ITL, TPOT, 처리량과 여러 입력 분포를 측정할 수 있습니다. 비교 도구가 무엇이든 클라이언트가 병목이 되지 않는지도 확인해야 합니다.

### 흔한 잘못된 벤치마크

- 요청 하나의 tokens/s만 보고 서버 처리량을 결론 내림
- 한 엔진은 4비트, 다른 엔진은 BF16으로 비교함
- 짧고 동일한 프롬프트만 반복해 prefix cache 이점을 과대평가함
- `max_tokens`만 지정하고 실제 생성 길이를 통제하지 않음
- 무한 요청률 결과를 실제 사용자 지연으로 해석함
- 서버 시작 직후 모델 로딩 시간을 정상 요청 지연에 섞음
- 평균값만 보고 p99와 실패 요청을 제외함

## 프로덕션 아키텍처에서 빠지기 쉬운 것

~~~text
클라이언트
  → API Gateway: TLS, 인증, rate limit, request ID
  → Model Router: 모델·LoRA·캐시 친화적 라우팅
  → Serving Replicas: vLLM / SGLang / 기타 엔진
  → GPU·가속기

별도 제어 계층
  ├─ 모델 저장소와 버전 관리
  ├─ 배포·롤백·오토스케일링
  ├─ 로그·메트릭·트레이싱
  └─ 평가·보안·비용 대시보드
~~~

### 보안

- 엔진 포트를 인터넷에 직접 공개하지 않습니다.
- 인증과 사용자·조직별 권한을 게이트웨이에서 검사합니다.
- prompt와 output 로그에 개인정보·비밀이 남지 않도록 마스킹과 보존 기간을 정합니다.
- Hugging Face token과 저장소 자격 증명은 secret manager로 전달합니다.
- `trust_remote_code`는 모델 저장소의 코드를 실행할 수 있으므로 revision 검증과 격리가 필요합니다.
- 모델 라이선스가 상업적 사용, 재배포와 생성물 정책에 맞는지 확인합니다.
- 관리·메트릭·프로파일링 endpoint도 내부망과 권한으로 보호합니다.

### 과부하와 backpressure

대기열을 무한히 늘리면 요청을 받는 순간에는 성공처럼 보여도 TTFT가 계속 증가하다 전체 서비스가 무너집니다. 최대 동시 요청, 대기열 길이, timeout과 거절 정책을 명시해야 합니다. 재시도는 지수 backoff와 jitter를 사용하고, 이미 과부하인 서버에 즉시 재시도 폭풍을 만들지 않아야 합니다.

### 오토스케일링

GPU 사용률만으로 확장하면 늦을 수 있습니다. KV 캐시가 메모리를 차지해도 연산 사용률은 낮을 수 있고, 긴 prefill이 대기열을 만들기도 합니다. 다음 신호를 함께 봅니다.

- 대기 중·실행 중 요청 수
- 예약·사용 중인 KV cache block
- TTFT와 ITL의 p95·p99
- 입력·출력 token rate
- 요청 거절과 OOM 횟수
- prefix cache hit rate
- 모델 로딩과 새 replica 준비 시간

LLM replica는 모델을 다운로드하고 GPU 메모리에 올리는 데 시간이 걸리므로 일반 웹 서버보다 scale-out이 느립니다. 예측 확장, 최소 warm replica와 로컬 모델 캐시를 고려해야 합니다.

### 관측 가능성

요청 ID를 게이트웨이부터 모델 서버까지 전달하고 다음을 기록합니다.

- 모델과 revision, quantization, adapter
- 입력·출력 토큰 수와 finish reason
- queue, prefill, decode와 전체 시간
- TTFT, TPOT, ITL, 처리량
- batch 크기와 scheduler 상태
- GPU 메모리, KV 캐시와 cache hit rate
- 오류 유형, 취소와 timeout

프롬프트 원문을 항상 남겨야 하는 것은 아닙니다. 민감도에 따라 길이, hash, 분류 결과 같은 최소 메타데이터만 기록하는 방식을 선택할 수 있습니다.

### 모델 교체와 롤백

같은 API라도 모델을 바꾸면 토크나이저, chat template, 도구 호출과 안전 특성이 달라집니다. 새 버전을 별도 replica에 올리고 준비 상태를 확인한 뒤 일부 트래픽만 보내는 canary 배포가 안전합니다. 품질 평가, 지연, 오류와 비용이 기준을 넘으면 즉시 이전 모델로 라우팅할 수 있어야 합니다.

## 상황별 선택 가이드

### 개인 PC에서 가장 빨리 시작하고 싶다

**Ollama**를 먼저 고려합니다. 모델 다운로드·실행·API가 단순하고 macOS, Windows와 Linux 개발 경험이 좋습니다. 세밀한 GGUF 옵션과 C/C++ 내장이 중요하면 llama.cpp를 직접 사용합니다.

### 단일 GPU 서버에 사내 챗 API를 만들고 싶다

**vLLM**을 기본 후보로 두고 **SGLang**을 같은 모델·트래픽으로 비교합니다. 일반적인 요청이 많고 폭넓은 통합이 중요하면 vLLM이 편리한 출발점입니다. 긴 접두사가 반복되거나 구조화 생성 비중이 높다면 SGLang을 적극적으로 시험합니다.

### 에이전트와 RAG에서 같은 문맥을 반복한다

**SGLang의 RadixAttention**이 잘 맞을 가능성이 큽니다. 단, 여러 replica를 쓴다면 prefix-aware routing이 없을 때 캐시 이점이 분산될 수 있으므로 라우터까지 함께 설계합니다.

### NVIDIA GPU에서 최고 수준의 맞춤 최적화가 필요하다

vLLM과 SGLang을 먼저 측정한 뒤 하드웨어 전용 튜닝과 NVIDIA 생태계가 중요하면 **TensorRT-LLM**도 비교합니다. 최고 성능은 모델, GPU와 정밀도별로 달라지므로 엔지니어링 비용까지 평가합니다.

### 여러 종류의 AI 모델 파이프라인을 운영한다

LLM 엔진 하나만으로 해결하려 하지 말고 **Triton, KServe 또는 Ray Serve** 같은 상위 계층을 검토합니다. LLM 생성은 vLLM·SGLang·TensorRT-LLM에 맡기고, embedding·reranker·전처리 모델과 라우팅을 플랫폼에서 묶을 수 있습니다.

### 이미 TGI를 안정적으로 운영 중이다

유지보수 모드라는 이유만으로 즉시 마이그레이션할 필요는 없습니다. 보안 수정, 지원 모델과 현재 SLO를 검토한 뒤 vLLM·SGLang으로 옮겼을 때 얻는 이익과 회귀 위험을 측정합니다. 신규 기능과 장기 로드맵이 중요하다면 단계적 이중 운영이 안전합니다.

## 도입 순서

1. 서비스의 모델, 최대 문맥과 품질 기준을 정합니다.
2. 실제 입력·출력 길이와 동시성 분포를 수집합니다.
3. 로컬 개발은 Ollama, 서버 후보는 vLLM·SGLang으로 작은 PoC를 만듭니다.
4. 동일한 가중치·정밀도·템플릿으로 성능과 품질을 비교합니다.
5. 인증, timeout, 대기열과 관측 계층을 먼저 붙입니다.
6. 목표 SLO를 만족하는 동시성과 GPU당 수용량을 찾습니다.
7. 장애, OOM, 요청 취소와 새 replica 기동을 시험합니다.
8. canary로 실제 트래픽 일부를 보내고 결과를 검증합니다.
9. 비용과 운영 복잡도를 포함해 엔진을 선택합니다.
10. 버전과 모델 revision을 고정하고 재현 가능한 배포로 만듭니다.

## 자주 묻는 질문

### Ollama를 프로덕션에서 쓰면 안 되는가

그렇지 않습니다. 소규모 내부 서비스나 단일 노드 제품에서 요구사항을 충족할 수 있습니다. 다만 인증, 동시성, 수평 확장, 장애 복구와 관측 요구를 별도 계층이 충족하는지 확인해야 합니다. 핵심은 제품 이름이 아니라 SLO와 운영 조건입니다.

### vLLM이 항상 SGLang보다 빠른가

아닙니다. 반대도 성립하지 않습니다. 공유 접두사, 입력·출력 길이, 동시성, 모델과 하드웨어에 따라 결과가 달라집니다. 동일 조건의 실제 워크로드로 측정해야 합니다.

### 양자화 모델이면 품질이 크게 떨어지는가

비트 수만으로 답할 수 없습니다. 양자화 알고리즘, calibration, 모델 크기와 작업에 따라 차이가 큽니다. 일반 대화 점수만 보지 말고 코드, 도구 호출, JSON 유효성, 검색 근거 인용처럼 서비스에 중요한 항목을 평가해야 합니다.

### GPU 메모리에 모델이 들어가면 동시 요청도 처리할 수 있는가

모델 가중치가 들어가는 것은 최소 조건입니다. KV 캐시, 커널 작업 공간, CUDA graph와 순간 활성값을 위한 여유가 필요합니다. 긴 문맥과 높은 동시성은 KV 캐시를 크게 늘립니다.

### OpenAI 호환이면 코드 변경 없이 교체할 수 있는가

기본 chat completions는 `base_url` 변경만으로 동작할 수 있습니다. 하지만 구조화 출력, tool calling, streaming, usage와 오류 처리는 차이가 있을 수 있으므로 contract test가 필요합니다.

## 마무리

LLM 서빙의 본질은 모델 파일에 HTTP API를 붙이는 것이 아닙니다. 서로 다른 길이의 요청을 계속 재배치하고, 제한된 가속기 메모리에 KV 캐시를 효율적으로 저장하며, 첫 토큰 지연과 전체 처리량 사이의 균형을 지키는 시스템 문제입니다.

- **Ollama**는 로컬에서 모델을 다루는 복잡성을 크게 줄입니다.
- **vLLM**은 폭넓은 모델 생태계와 범용 고성능 서버의 균형이 좋습니다.
- **SGLang**은 접두사 재사용과 복잡한 생성·분산 워크로드에서 강력한 최적화를 제공합니다.

최종 선택은 기능 개수나 공개 벤치마크 순위가 아니라 **내 모델, 내 하드웨어, 내 요청 분포와 내 SLO**로 결정해야 합니다. 엔진과 함께 게이트웨이, 라우팅, 관측, 보안과 배포 전략까지 설계했을 때 비로소 추론 코드가 안정적인 AI 서비스가 됩니다.

## 참고 자료

- [Ollama 공식 문서](https://docs.ollama.com/)
- [Ollama API 문서](https://docs.ollama.com/api)
- [vLLM 공식 문서](https://docs.vllm.ai/en/latest/)
- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
- [SGLang 공식 문서](https://docs.sglang.ai/)
- [SGLang: Efficient Execution of Structured Language Model Programs](https://arxiv.org/abs/2312.07104)
- [llama.cpp 공식 저장소](https://github.com/ggml-org/llama.cpp)
- [TensorRT-LLM 공식 문서](https://nvidia.github.io/TensorRT-LLM/)
- [Hugging Face TGI 공식 문서](https://huggingface.co/docs/text-generation-inference/en/index)
- [NVIDIA Triton Inference Server 공식 문서](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)
