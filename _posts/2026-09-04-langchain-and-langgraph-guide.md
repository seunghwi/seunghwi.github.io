---
layout: single
title: "LangChain과 LangGraph 완전 이해: 개념부터 에이전트 워크플로까지"
categories:
  - "AI"
permalink: /topics/ai/langchain-and-langgraph-guide/
tags:
  - "AI"
  - "LLM"
  - "LangChain"
  - "LangGraph"
  - "AI 에이전트"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

LLM API를 한 번 호출하는 코드는 간단합니다. 하지만 실제 서비스를 만들려면 대화 기록, 프롬프트, 도구 호출, 검색 결과, 출력 검증, 오류 처리와 실행 상태를 함께 관리해야 합니다. 작업이 여러 단계로 늘어나면 “모델에 무엇을 보여주고, 다음에 무엇을 실행하며, 실패한 지점에서 어떻게 다시 시작할 것인가”가 핵심 문제가 됩니다.

**LangChain**은 모델·메시지·도구·검색·구조화 출력과 에이전트를 공통 인터페이스로 조립하도록 돕습니다. **LangGraph**는 그 실행 과정을 상태가 있는 그래프로 표현해 분기, 반복, 병렬 실행, 중단과 재개를 직접 제어하도록 돕습니다.

이 글은 Python과 LangChain v1 계열 API를 기준으로 두 프로젝트의 개념과 관계를 설명합니다. 예제의 모델 ID는 글 작성 시점의 예시이므로 사용하는 제공자와 계정에서 지원하는 모델로 바꿔야 합니다.

LangChain과 LangGraph는 API가 빠르게 발전하는 프로젝트입니다. 오래된 글에서 `LLMChain`, `ConversationChain`, `initialize_agent` 또는 `langgraph.prebuilt.create_react_agent`를 발견할 수 있지만, 현재 LangChain v1에서는 `create_agent`가 표준 에이전트 생성 API입니다. 코드를 적용하기 전에 [공식 마이그레이션 문서](https://docs.langchain.com/oss/python/migrate/langchain-v1)를 함께 확인하세요.
{: .notice--warning}

## 가장 먼저 보는 관계도

~~~text
애플리케이션
  │
  ├─ LangChain
  │    ├─ 모델과 메시지의 공통 인터페이스
  │    ├─ 프롬프트·도구·구조화 출력
  │    ├─ 검색과 외부 시스템 연동
  │    └─ create_agent + middleware
  │                  │
  │                  └─ 내부 실행 기반: LangGraph
  │
  └─ LangGraph
       ├─ State: 실행 중 공유할 데이터
       ├─ Node: 수행할 단계
       ├─ Edge: 다음 단계와 조건
       ├─ Checkpointer: 상태 저장과 재개
       └─ Interrupt·Command·Streaming
~~~

두 라이브러리는 서로 대체하는 관계가 아닙니다.

- 간단한 모델 호출이나 일반적인 도구 사용 에이전트는 LangChain만으로 시작할 수 있습니다.
- 실행 순서와 상태 전이를 세밀하게 제어하려면 LangGraph를 직접 사용합니다.
- LangGraph 안에서 LangChain의 모델과 도구를 사용할 수 있습니다.
- LangGraph는 LangChain 없이 일반 Python 함수만으로도 사용할 수 있습니다.
- LangChain의 [`create_agent`](https://docs.langchain.com/oss/python/langchain/agents)는 LangGraph 기반의 에이전트 런타임을 만듭니다.

## 설치와 실행 준비

기본 패키지와 OpenAI 연동을 예로 들면 다음과 같이 설치합니다.

~~~bash
python -m pip install -U langchain langgraph langchain-openai
~~~

API 키는 소스 코드에 직접 적지 않고 환경 변수나 비밀 관리 시스템으로 전달합니다.

~~~bash
export OPENAI_API_KEY="발급받은_API_키"
~~~

다른 제공자를 사용한다면 해당 통합 패키지를 설치합니다. 예를 들어 Anthropic 모델은 `langchain-anthropic`, Google 계열은 지원하는 공식 LangChain 통합 패키지가 필요합니다. 제공자마다 모델 기능과 파라미터가 다르므로 [통합 목록](https://docs.langchain.com/oss/python/integrations/chat/index)에서 도구 호출, 구조화 출력과 멀티모달 지원 여부를 확인합니다.

## LangChain이 해결하는 문제

모델 제공자의 API는 메시지 형식, 도구 호출 구조, 스트리밍 이벤트와 응답 메타데이터가 서로 다릅니다. 제공자를 바꿀 때 애플리케이션 전체를 다시 작성한다면 실험과 유지보수가 어려워집니다.

LangChain은 다음 요소를 조립 가능한 공통 구성 요소로 다룹니다.

| 구성 요소 | 역할 | 대표적인 사용 |
|---|---|---|
| Model | LLM 호출 인터페이스 | 응답 생성, 도구 선택, 구조화 출력 |
| Message | 대화의 역할·내용·메타데이터 | system, user, assistant, tool 메시지 |
| Prompt | 입력을 일관된 메시지로 변환 | 변수 삽입, 역할별 지시 구성 |
| Tool | 모델이 요청할 수 있는 외부 함수 | 검색, 계산, DB, 사내 API |
| Structured output | 응답을 정해진 스키마로 제한 | Pydantic 객체, JSON, 데이터 추출 |
| Retriever | 질문과 관련된 자료 검색 | RAG, 문서 질의응답 |
| Agent | 모델과 도구를 반복 실행 | 조사, 지원, 업무 자동화 |
| Middleware | 에이전트 실행 전후 개입 | 요약, 로깅, 가드레일, 동적 모델 선택 |
| Memory | 실행 상태 보존 | 대화별 단기 기억, 사용자별 장기 기억 |

LangChain의 목적은 모델 자체를 학습시키는 것이 아닙니다. 이미 존재하는 모델과 데이터·도구·애플리케이션 로직 사이를 연결하는 **애플리케이션 프레임워크**입니다.

## LangChain의 기본 구성 요소

### Model: 추론 엔진을 공통 방식으로 호출하기

채팅 모델은 메시지를 입력받아 `AIMessage`를 반환합니다. 가장 작은 호출은 다음과 같습니다.

~~~python
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-5.4")

response = model.invoke([
    {"role": "system", "content": "핵심만 설명하는 Python 튜터입니다."},
    {"role": "user", "content": "리스트 컴프리헨션을 한 문장으로 설명해줘."},
])

print(response.text)
~~~

`invoke`는 응답이 완성될 때까지 기다립니다. 긴 출력은 `stream`, 서로 독립적인 여러 입력은 `batch` 또는 비동기 API를 사용할 수 있습니다. 지원 범위는 모델 통합마다 다릅니다.

모델 이름 문자열은 편리하지만, timeout이나 제공자 전용 옵션을 세밀하게 지정하려면 모델 클래스를 직접 만듭니다.

~~~python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-5.4",
    temperature=0,
    timeout=30,
    max_retries=2,
)
~~~

`temperature=0`이어도 모든 제공자가 완전히 동일한 결과를 보장하는 것은 아닙니다. 모델 버전, 서버 구현과 병렬 계산에 따라 출력이 달라질 수 있으므로 중요한 로직은 자연어 일치가 아니라 스키마와 테스트로 검증해야 합니다.
{: .notice--info}

### Message: 문자열보다 많은 대화 정보

[Message](https://docs.langchain.com/oss/python/langchain/messages)는 단순한 문자열이 아니라 역할, 콘텐츠와 메타데이터를 담습니다.

- `SystemMessage`: 모델의 역할과 기본 규칙
- `HumanMessage`: 사용자의 입력
- `AIMessage`: 모델 응답, 도구 호출과 사용량 정보
- `ToolMessage`: 도구 실행 결과

~~~python
from langchain.messages import HumanMessage, SystemMessage

messages = [
    SystemMessage("답변은 한국어로 작성하세요."),
    HumanMessage("LangChain에서 메시지가 필요한 이유는?"),
]

answer = model.invoke(messages)
print(answer.text)
~~~

현대 모델의 메시지는 텍스트만 담지 않을 수 있습니다. 이미지, 오디오, 문서, 추론 요약, 인용과 제공자별 콘텐츠 블록도 포함할 수 있습니다. LangChain은 이런 차이를 공통 메시지 구조로 다룰 수 있게 합니다.

### Prompt Template: 반복 가능한 입력 만들기

문자열을 직접 이어 붙이면 변수 누락과 역할 구분 오류가 생기기 쉽습니다. `ChatPromptTemplate`은 입력 값을 역할별 메시지로 변환합니다.

~~~python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 {language} 코드 리뷰어입니다."),
    ("user", "다음 코드의 문제를 세 가지 이내로 설명하세요:\n{code}"),
])

messages = prompt.invoke({
    "language": "Python",
    "code": "items = [1, 2, 3]\nprint(items[3])",
})

print(model.invoke(messages).text)
~~~

프롬프트는 보안 경계가 아닙니다. “절대 비밀을 출력하지 마라”라는 문장만으로 데이터 접근을 막을 수 없습니다. 모델에 전달할 데이터 자체를 권한에 따라 필터링해야 합니다.

### Runnable과 파이프 연산자

LangChain의 많은 구성 요소는 입력을 받아 출력을 만드는 `Runnable` 인터페이스를 따릅니다. 프롬프트의 출력이 모델의 입력이 되므로 파이프 연산자로 연결할 수 있습니다.

~~~python
chain = prompt | model

answer = chain.invoke({
    "language": "Python",
    "code": "value = None\nprint(value.upper())",
})

print(answer.text)
~~~

이런 고정 파이프라인은 순서가 명확하고 디버깅하기 쉽습니다. 모델이 도구를 스스로 고르고 여러 번 반복해야 할 때는 agent가, 명시적인 분기와 상태가 필요할 때는 LangGraph가 더 적합합니다.

### Structured output: 자연어를 애플리케이션 데이터로 바꾸기

모델에게 “JSON으로 답해”라고 요청하는 것만으로는 필드 누락, 잘못된 타입과 JSON 문법 오류를 막기 어렵습니다. [구조화 출력](https://docs.langchain.com/oss/python/langchain/structured-output)은 Pydantic, `TypedDict` 또는 JSON Schema로 원하는 결과를 정의합니다.

~~~python
from pydantic import BaseModel, Field


class Ticket(BaseModel):
    category: str = Field(description="문의 분류")
    urgent: bool = Field(description="긴급 여부")
    summary: str = Field(description="한 문장 요약")


extractor = model.with_structured_output(Ticket)

ticket = extractor.invoke(
    "결제가 두 번 됐습니다. 오늘 안에 확인해 주세요."
)

print(ticket.category)
print(ticket.urgent)
print(ticket.summary)
~~~

제공자가 네이티브 JSON schema를 지원하면 그 기능을 사용할 수 있고, 그렇지 않으면 도구 호출 방식으로 구조를 강제할 수 있습니다. Pydantic 검증이 통과했다는 것은 **형식이 맞다**는 뜻이지 내용이 사실이라는 뜻은 아닙니다.

### Tool: 모델과 실제 세계 사이의 함수

도구는 이름, 설명, 인자 스키마와 실행 함수를 묶습니다. 모델은 도구 실행을 요청할 뿐이고 실제 함수는 애플리케이션이 실행합니다.

~~~python
from langchain.tools import tool


@tool
def calculate_vat(price: int, rate: float = 0.1) -> float:
    """상품 가격과 세율을 받아 부가세를 계산한다."""
    return price * rate
~~~

타입 힌트는 인자 스키마가 되고 docstring은 모델이 도구를 선택할 때 읽는 설명이 됩니다. 도구 이름과 설명이 모호하면 모델이 잘못된 도구를 선택하거나 잘못된 값을 넣기 쉽습니다.

직접 모델에 도구를 묶으면 선택과 실행 루프를 애플리케이션이 관리해야 합니다.

~~~python
model_with_tools = model.bind_tools([calculate_vat])

reply = model_with_tools.invoke("15,000원 상품의 부가세를 계산해줘.")
print(reply.tool_calls)
~~~

이 코드는 도구 호출 **요청**을 확인하는 단계입니다. `reply.tool_calls`의 함수를 실행하고 `tool_call_id`가 일치하는 `ToolMessage`를 다시 모델에 보내야 최종 자연어 답을 받을 수 있습니다. 일반적인 반복은 agent가 대신 처리합니다.

## LangChain Agent: 모델과 도구를 반복 실행하기

에이전트의 핵심 루프는 단순합니다.

~~~text
사용자 입력
   ↓
모델 호출 ── 최종 답변 ──→ 종료
   │
   └─ 도구 호출 요청
          ↓
       도구 실행
          ↓
       결과를 메시지에 추가
          └──────────────→ 모델 다시 호출
~~~

[공식 Agent 문서](https://docs.langchain.com/oss/python/langchain/agents)의 `create_agent`는 이 루프를 LangGraph로 구성합니다.

### 간단한 계산 에이전트

~~~python
from langchain.agents import create_agent
from langchain.tools import tool


@tool
def multiply(a: int, b: int) -> int:
    """두 정수를 곱한다."""
    return a * b


agent = create_agent(
    model="openai:gpt-5.4",
    tools=[multiply],
    system_prompt="계산이 필요하면 제공된 도구를 사용하세요.",
)

result = agent.invoke({
    "messages": [
        {"role": "user", "content": "37과 29를 곱하면 얼마야?"}
    ]
})

print(result["messages"][-1].text)
~~~

실행 과정은 대략 다음과 같습니다.

1. 모델이 `multiply(a=37, b=29)` 호출을 만듭니다.
2. agent가 Python 함수를 실행합니다.
3. 결과 `1073`을 `ToolMessage`로 추가합니다.
4. 모델이 도구 결과를 근거로 최종 문장을 만듭니다.

모델이 계산을 직접 해도 답할 수 있지만 예제는 도구 루프의 구조를 보여주기 위한 것입니다. 실제로는 환율 API, 상품 DB, 사내 검색과 주문 시스템 같은 외부 기능을 연결합니다.

### 간단한 검색 도구를 추가하기

외부 벡터 DB 없이 RAG의 모양만 확인하는 예제입니다.

~~~python
from langchain.tools import tool

POLICIES = {
    "환불": "구매 후 7일 이내, 사용하지 않은 상품은 환불할 수 있습니다.",
    "배송": "평일 오후 2시 이전 주문은 당일 발송합니다.",
}


@tool
def search_policy(keyword: str) -> str:
    """환불 또는 배송 정책을 키워드로 검색한다."""
    return POLICIES.get(keyword, "관련 정책을 찾지 못했습니다.")


support_agent = create_agent(
    model="openai:gpt-5.4",
    tools=[search_policy],
    system_prompt=(
        "고객 지원 담당자입니다. 정책 질문에는 반드시 "
        "search_policy 결과만 근거로 답하세요."
    ),
)
~~~

실제 RAG에서는 문서를 일정 크기로 나누고, 임베딩과 키워드로 후보를 검색하고, reranker로 다시 정렬한 뒤 관련 구간을 모델에 전달합니다. 검색 권한과 출처 표시, 찾지 못했을 때 답하지 않는 정책도 함께 설계해야 합니다.

### Middleware: 에이전트 루프에 개입하기

LangChain v1의 middleware는 모델·도구 호출 전후에 실행되는 확장 지점입니다. 동적 system prompt, 대화 요약, 도구 선택 제한, 재시도, 로깅, 개인정보 검사와 human-in-the-loop를 핵심 agent 코드를 바꾸지 않고 추가할 수 있습니다.

~~~python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-5.4",
    tools=[search_policy],
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-5.4-mini",
            trigger={"tokens": 4000},
            keep={"messages": 20},
        )
    ],
)
~~~

이 middleware는 대화가 기준을 넘으면 오래된 메시지를 요약해 상태를 갱신하고 최근 메시지는 유지합니다. 요약 과정에서도 중요한 조건이 사라질 수 있으므로 보존해야 할 사실을 별도 상태나 장기 저장소에 분리하는 편이 안전합니다.

LangChain 문서는 신뢰성의 핵심을 **context engineering**으로 설명합니다. 모델 호출마다 필요한 지시, 메시지, 도구와 출력 형식을 알맞게 제공하고, 도구가 접근할 상태와 저장소를 통제하는 작업입니다.

## LangGraph가 필요한 이유

일반적인 agent loop만으로 충분하지 않은 상황을 생각해 봅니다.

- 결제 전에는 반드시 사람의 승인을 받아야 합니다.
- 문서 초안이 기준을 통과하지 못하면 최대 세 번 다시 작성해야 합니다.
- 서로 독립적인 조사 작업 다섯 개를 병렬로 실행해야 합니다.
- 서버가 재시작돼도 몇 시간짜리 작업을 마지막 성공 단계부터 이어야 합니다.
- 사용자별 대화 상태를 서로 섞이지 않게 저장해야 합니다.
- 실행 도중 상태를 확인하고 특정 값을 수정한 뒤 재개해야 합니다.

이 요구는 “모델이 알아서 하도록” 프롬프트에 적는 것보다 코드의 제어 흐름으로 강제해야 합니다. [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)는 장시간 실행되는 상태 기반 workflow와 agent를 위한 저수준 오케스트레이션 프레임워크입니다.

## LangGraph의 세 가지 중심 개념

[Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)는 그래프를 `State`, `Node`, `Edge`로 설명합니다.

### State: 현재 실행의 공유 데이터

State는 그래프가 현재 알고 있는 값의 스냅샷입니다. 보통 `TypedDict`, dataclass 또는 Pydantic 모델로 스키마를 정의합니다.

~~~python
from typing_extensions import TypedDict


class ArticleState(TypedDict):
    topic: str
    outline: list[str]
    draft: str
    score: int
~~~

모든 값을 하나의 거대한 dict에 넣기보다 노드가 실제로 공유해야 하는 데이터만 명시합니다. 상태 스키마가 명확하면 체크포인트, 디버깅, 병렬 업데이트와 테스트가 쉬워집니다.

### Node: 상태를 읽고 업데이트를 반환하는 단계

Node는 일반 Python 함수 또는 비동기 함수입니다. 현재 state를 받고 전체 state가 아니라 **변경할 값**을 반환합니다.

~~~python
def make_outline(state: ArticleState):
    topic = state["topic"]
    return {
        "outline": [
            f"{topic}의 정의",
            f"{topic}의 작동 방식",
            f"{topic}의 주의점",
        ]
    }
~~~

노드 안에서 LLM을 호출할 수도 있고, DB를 읽거나 순수한 규칙 검사를 수행할 수도 있습니다. 모든 노드가 agent일 필요는 없습니다.

### Edge: 다음에 실행할 단계

Edge는 노드 사이의 이동을 정의합니다.

- 일반 edge는 항상 같은 다음 노드로 갑니다.
- conditional edge는 state를 보고 다음 노드를 선택합니다.
- `START`와 `END`는 그래프의 가상 시작점과 종료점입니다.

## 첫 번째 LangGraph 예제: 분기하는 글 검수 흐름

아래 예제는 LLM 없이 그래프의 구조만 보여줍니다. 글 길이가 기준 이상이면 승인하고, 짧으면 보강합니다.

~~~python
from typing import Literal
from typing_extensions import TypedDict
from langgraph.graph import END, START, StateGraph


class ReviewState(TypedDict):
    draft: str
    result: str


def inspect_length(state: ReviewState):
    length = len(state["draft"])
    return {"result": "pass" if length >= 30 else "retry"}


def expand_draft(state: ReviewState):
    return {"draft": state["draft"] + " 핵심 원리와 예제를 추가합니다."}


def route_after_review(state: ReviewState) -> Literal["expand", END]:
    if state["result"] == "pass":
        return END
    return "expand"


builder = StateGraph(ReviewState)
builder.add_node("review", inspect_length)
builder.add_node("expand", expand_draft)

builder.add_edge(START, "review")
builder.add_conditional_edges("review", route_after_review)
builder.add_edge("expand", "review")

graph = builder.compile()
result = graph.invoke({"draft": "LangGraph 소개", "result": ""})

print(result)
~~~

흐름은 다음과 같습니다.

~~~text
START → review ── pass ──→ END
            │
            └─ retry → expand ──→ review
~~~

반복 그래프에는 반드시 종료 조건이 있어야 합니다. LLM 평가가 계속 같은 결과를 내거나 외부 API가 실패하면 무한 반복할 수 있으므로 최대 시도 횟수, 시간 제한과 `recursion_limit`을 함께 둡니다.

## State update와 Reducer

기본적으로 노드가 반환한 값은 해당 state key의 이전 값을 덮어씁니다. 여러 노드의 결과를 누적하려면 reducer를 지정합니다.

~~~python
import operator
from typing import Annotated
from typing_extensions import TypedDict


class ResearchState(TypedDict):
    query: str
    notes: Annotated[list[str], operator.add]
~~~

`notes`를 업데이트하는 노드가 `{"notes": ["새 메모"]}`를 반환하면 `operator.add`가 기존 목록 뒤에 새 목록을 붙입니다. reducer가 없으면 목록 전체가 새 값으로 교체됩니다.

메시지 목록에는 단순 `operator.add` 대신 메시지 ID를 기준으로 추가·갱신하고 dict를 Message 객체로 변환하는 `add_messages` reducer 또는 `MessagesState`를 사용합니다.

~~~python
from langgraph.graph import MessagesState


def answer(state: MessagesState):
    reply = model.invoke(state["messages"])
    return {"messages": [reply]}
~~~

Reducer는 병렬 노드가 같은 key를 업데이트할 때 특히 중요합니다. 어떤 결과를 합치고 어떤 값은 덮어쓸지 정하지 않으면 상태 충돌이 발생합니다.

## LLM을 넣은 LangGraph 예제

간단한 초안 생성과 검토 workflow를 구성해 봅니다.

~~~python
from typing import Literal
from typing_extensions import TypedDict
from langchain.chat_models import init_chat_model
from langgraph.graph import END, START, StateGraph

model = init_chat_model("openai:gpt-5.4")


class WritingState(TypedDict):
    topic: str
    draft: str
    feedback: str
    attempts: int


def write(state: WritingState):
    prompt = f"""
    주제: {state['topic']}
    이전 피드백: {state.get('feedback', '')}
    300자 이내의 설명을 작성하세요.
    """
    return {
        "draft": model.invoke(prompt).text,
        "attempts": state.get("attempts", 0) + 1,
    }


def review(state: WritingState):
    prompt = f"""
    다음 글이 초보자에게 명확한지 평가하세요.
    문제가 없으면 정확히 OK만, 문제가 있으면 한 문장 피드백만 출력하세요.

    {state['draft']}
    """
    return {"feedback": model.invoke(prompt).text.strip()}


def decide(state: WritingState) -> Literal["write", END]:
    if state["feedback"] == "OK" or state["attempts"] >= 3:
        return END
    return "write"


builder = StateGraph(WritingState)
builder.add_node("write", write)
builder.add_node("review", review)
builder.add_edge(START, "write")
builder.add_edge("write", "review")
builder.add_conditional_edges("review", decide)

workflow = builder.compile()

result = workflow.invoke({
    "topic": "LangGraph의 상태",
    "draft": "",
    "feedback": "",
    "attempts": 0,
})

print(result["draft"])
print(result["feedback"])
~~~

이 패턴을 evaluator-optimizer라고 부를 수 있습니다. 생성 노드와 평가 노드를 분리하면 각 프롬프트와 모델을 독립적으로 테스트하고, 평가 실패 시 재시도 횟수를 코드로 제한할 수 있습니다.

이 예제의 `OK` 문자열 비교는 설명을 위한 단순화입니다. 실제 서비스에서는 `Literal["pass", "retry"]`가 들어간 구조화 출력을 사용해야 공백이나 표현 차이로 분기가 잘못되는 문제를 줄일 수 있습니다.
{: .notice--info}

## Persistence: 그래프 상태 저장하기

LangGraph는 checkpointer를 사용해 각 실행 단계의 state snapshot을 저장합니다. [Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)가 활성화되면 다음 기능의 기반이 됩니다.

- 같은 대화 thread의 단기 기억
- 실패한 단계에서 재개하는 durable execution
- 사람이 확인하고 승인하는 human-in-the-loop
- 이전 checkpoint를 재생하거나 다른 경로로 분기하는 time travel
- 장시간 작업의 장애 복구

개발 중에는 메모리 checkpointer로 동작을 확인할 수 있습니다.

~~~python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
workflow = builder.compile(checkpointer=checkpointer)

config = {
    "configurable": {
        "thread_id": "article-42"
    }
}

result = workflow.invoke(
    {
        "topic": "상태 저장",
        "draft": "",
        "feedback": "",
        "attempts": 0,
    },
    config,
)
~~~

`thread_id`는 어느 실행의 checkpoint를 읽고 쓸지 구분하는 열쇠입니다. 사용자 ID와 같다고 가정하지 말고, 대화·작업 단위로 충돌하지 않는 값을 만들고 접근 권한을 별도로 검사해야 합니다.

`InMemorySaver`는 프로세스가 종료되면 사라지므로 테스트용입니다. 운영 환경에서는 PostgreSQL 등 영속 저장소 기반 checkpointer를 사용하고 백업, 암호화, 보존 기간과 개인정보 삭제 정책을 함께 설계합니다.

## Interrupt: 사람의 승인을 기다렸다 재개하기

결제, 이메일 발송과 파일 삭제처럼 되돌리기 어려운 작업은 모델 판단만으로 실행하지 않는 편이 안전합니다. LangGraph의 [`interrupt`](https://docs.langchain.com/oss/python/langgraph/interrupts)는 노드 실행을 멈추고 JSON 직렬화 가능한 정보를 호출자에게 돌려줍니다.

~~~python
from typing_extensions import TypedDict
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import END, START, StateGraph
from langgraph.types import Command, interrupt


class PaymentState(TypedDict):
    amount: int
    approved: bool


def request_approval(state: PaymentState):
    approved = interrupt({
        "question": f"{state['amount']:,}원을 결제할까요?",
        "amount": state["amount"],
    })
    return {"approved": bool(approved)}


builder = StateGraph(PaymentState)
builder.add_node("approval", request_approval)
builder.add_edge(START, "approval")
builder.add_edge("approval", END)

payment_graph = builder.compile(checkpointer=InMemorySaver())
config = {"configurable": {"thread_id": "payment-1001"}}

# 첫 호출은 interrupt에서 멈춘다.
paused = payment_graph.invoke(
    {"amount": 50000, "approved": False},
    config,
)
print(paused["__interrupt__"])

# 사용자가 승인한 뒤 같은 thread를 재개한다.
finished = payment_graph.invoke(Command(resume=True), config)
print(finished["approved"])
~~~

재개할 때 노드는 처음부터 다시 실행되며 `interrupt()`는 전달받은 resume 값을 반환합니다. 따라서 interrupt 전에 수행하는 파일 쓰기, 결제 요청과 메시지 발송 같은 side effect는 재실행돼도 안전한 **idempotent** 동작으로 만들거나 interrupt 뒤로 옮겨야 합니다.

## Command: 상태 갱신과 이동을 함께 표현하기

conditional edge는 state를 읽고 목적지만 선택합니다. 한 노드에서 state를 바꾸면서 이동할 곳도 결정해야 한다면 `Command`를 반환할 수 있습니다.

~~~python
from typing import Literal
from langgraph.types import Command


def classify(
    state: ReviewState,
) -> Command[Literal["manual_review", "publish"]]:
    if "개인정보" in state["draft"]:
        return Command(
            update={"result": "needs_review"},
            goto="manual_review",
        )

    return Command(
        update={"result": "safe"},
        goto="publish",
    )
~~~

반환 타입에 가능한 node 이름을 적으면 그래프 시각화와 정적 이해에 도움이 됩니다. 같은 node에 일반 edge도 추가해 둔 상태에서 `Command(goto=...)`를 반환하면 두 경로가 모두 실행될 수 있으므로 중복 edge를 주의합니다.

## Send와 병렬 작업

조사할 하위 주제 수가 실행 중에 결정되는 map-reduce 흐름에는 `Send`를 사용할 수 있습니다.

~~~python
from langgraph.types import Send


def dispatch_topics(state):
    return [
        Send("research_one", {"topic": topic})
        for topic in state["topics"]
    ]
~~~

각 `research_one` 노드는 서로 다른 입력으로 병렬 실행될 수 있고, reducer가 결과를 하나의 목록으로 모읍니다. 외부 API의 동시 요청 한도, 결과 순서, 일부 실패와 재시도 정책을 별도로 설계해야 합니다.

## Subgraph: 복잡한 흐름을 작은 그래프로 나누기

하나의 그래프가 커지면 조사, 작성, 승인과 발송 영역을 subgraph로 분리할 수 있습니다. 각 subgraph는 자체 state와 node를 가질 수 있고 부모 그래프와 필요한 값만 공유합니다.

Subgraph가 유용한 경우는 다음과 같습니다.

- 팀별로 workflow의 일부를 독립 개발할 때
- 여러 상위 workflow에서 같은 처리 흐름을 재사용할 때
- 서로 다른 agent에게 별도 상태와 도구를 부여할 때
- 내부 단계를 숨기고 명확한 입력·출력 경계를 만들 때

단순히 파일을 나누기 위해 subgraph를 만들기보다 독립적인 책임과 상태 경계가 있을 때 사용합니다.

## Streaming: 최종 답을 기다리지 않고 진행 상황 보여주기

LLM의 토큰만 스트리밍하는 것과 graph 상태 변화를 스트리밍하는 것은 다릅니다. LangGraph는 메시지 토큰, node별 state update, 전체 state snapshot과 사용자 정의 이벤트 등을 구분해 전달할 수 있습니다.

~~~python
for chunk in workflow.stream(
    {
        "topic": "LangGraph streaming",
        "draft": "",
        "feedback": "",
        "attempts": 0,
    },
    stream_mode="updates",
):
    print(chunk)
~~~

화면에는 “자료 검색 중”, “초안 검토 중” 같은 단계 진행을 표시하고, 로그에는 node 실행 시간과 오류를 기록할 수 있습니다. 스트림 연결이 끊겨도 실행 자체를 계속할지 취소할지 정책을 정해야 합니다.

## LangChain과 LangGraph 비교

| 기준 | LangChain | LangGraph |
|---|---|---|
| 추상화 수준 | 상대적으로 높음 | 낮고 명시적임 |
| 중심 관심사 | 모델·도구·검색과 agent 조립 | 상태 기반 실행과 제어 흐름 |
| 기본 진입점 | `create_agent`, model, tool | `StateGraph` |
| 실행 구조 | 일반적인 agent loop를 빠르게 구성 | node와 edge를 직접 정의 |
| 상태 관리 | agent state와 middleware로 접근 | state schema와 reducer를 직접 설계 |
| 분기·반복 | agent가 도구 필요 여부를 판단 | 코드로 조건과 종료 경로를 강제 |
| 영속성 | checkpointer를 agent에 전달 | compile 시 checkpointer 연결 |
| 사람 승인 | middleware 또는 기반 graph 기능 | interrupt와 Command로 직접 구성 |
| 적합한 시작점 | 챗봇, RAG agent, 도구 agent | 승인 흐름, 장시간 작업, 복합 workflow |

### LangChain만 사용하면 좋은 경우

- 모델을 공통 인터페이스로 호출하려는 경우
- 프롬프트와 구조화 출력을 연결하는 고정 chain
- 도구 몇 개를 사용하는 일반적인 agent
- 표준 대화 메모리와 middleware로 충분한 경우
- 빠르게 기능을 검증하는 프로토타입

### LangGraph를 직접 사용하면 좋은 경우

- 실행 순서가 비즈니스 규칙으로 정해진 경우
- 조건 분기, 반복과 병렬 처리를 코드로 보장해야 하는 경우
- 작업을 중단하고 사람이 승인한 뒤 재개해야 하는 경우
- 몇 분에서 며칠 동안 이어지는 durable workflow
- 여러 agent와 subgraph가 역할을 나누는 경우
- 실패 지점의 상태를 저장하고 다시 실행해야 하는 경우

### 둘 다 사용하면 좋은 경우

실무에서는 LangGraph가 전체 workflow를 제어하고 각 node에서 LangChain 모델, prompt, retriever와 tool을 사용하는 조합이 자연스럽습니다.

~~~text
LangGraph workflow
  ├─ 입력 검증 node: 일반 Python
  ├─ 문서 검색 node: LangChain retriever
  ├─ 답변 생성 node: LangChain model + prompt
  ├─ 품질 검사 node: structured output
  └─ 승인 node: LangGraph interrupt
~~~

## 자주 사용하는 설계 패턴

### Prompt chain

정해진 순서로 prompt와 model을 한 번씩 실행합니다. 요약, 분류와 추출처럼 흐름이 고정된 작업에 적합합니다. 가장 단순한 해결책이므로 agent보다 먼저 검토합니다.

### Routing

입력 분류 결과에 따라 서로 다른 node, model 또는 tool로 보냅니다. 예를 들어 결제 문의는 주문 시스템으로, 기술 문의는 문서 검색으로 전달할 수 있습니다. 분류 결과는 구조화 출력으로 제한하는 편이 안전합니다.

### Orchestrator-worker

상위 node가 작업을 여러 하위 주제로 나누고 worker node가 병렬 처리한 뒤 결과를 합칩니다. 심층 조사, 여러 파일 분석과 보고서 작성에 유용합니다. 분할 결과가 겹치거나 빠지지 않는지 검사 단계가 필요합니다.

### Evaluator-optimizer

생성 결과를 별도 evaluator가 평가하고 기준을 만족하지 못하면 피드백과 함께 다시 생성합니다. 품질을 높일 수 있지만 종료 조건이 없으면 비용이 계속 증가합니다. evaluator 자체의 오류와 두 모델이 같은 편향을 공유할 가능성도 고려합니다.

### Human-in-the-loop

모델이 제안한 작업을 사람이 확인·수정·거절한 뒤 실행합니다. 송금, 외부 메시지, 계정 변경과 삭제처럼 영향이 큰 도구에는 승인 단계를 코드로 강제합니다.

## Memory를 정확히 구분하기

AI 애플리케이션에서 memory라는 단어는 여러 의미로 사용됩니다.

| 종류 | 범위 | 예시 | 저장 위치 |
|---|---|---|---|
| 모델 컨텍스트 | 한 번의 모델 호출 | 현재 prompt와 도구 결과 | 요청 안의 토큰 |
| 단기 메모리 | 하나의 thread | 대화 기록, 작업 진행 상태 | LangGraph checkpoint |
| 장기 메모리 | 여러 thread | 사용자 선호, 누적된 사실 | Store 또는 애플리케이션 DB |
| 모델 파라미터 | 학습 이후 공통 | 사전학습에서 배운 패턴 | 모델 가중치 |

LangChain agent의 [단기 메모리](https://docs.langchain.com/oss/python/langchain/short-term-memory)는 graph state와 checkpointer를 사용합니다. 같은 `thread_id`를 재사용하면 이전 상태를 이어갈 수 있습니다.

장기 메모리는 모든 과거 대화를 prompt에 계속 넣는 방식이 아닙니다. 저장할 가치가 있는 사실을 추출하고 사용자·조직별 namespace에 저장한 뒤 필요한 순간에 검색해야 합니다. 잘못 추출된 기억을 수정하고 삭제할 기능도 필요합니다.

## 운영 환경에서 꼭 확인할 점

### 단순한 workflow부터 시작하기

분류 규칙과 고정 순서로 해결되는 문제에 자유로운 agent를 사용하면 비용과 실패 경로만 늘어날 수 있습니다. 먼저 일반 코드로 가능한 부분을 고정하고, 판단이 필요한 좁은 지점에만 LLM을 사용합니다.

### 상태를 작고 명확하게 유지하기

거대한 원문과 모든 중간 결과를 state에 계속 누적하면 checkpoint 크기와 모델 입력 비용이 커집니다. 원본은 외부 저장소에 두고 state에는 ID와 필요한 요약만 저장하는 방식을 고려합니다.

### 종료 조건과 예산 두기

최대 model call 횟수, graph recursion limit, 전체 시간, 토큰과 비용 한도를 둡니다. 모델이 “완료했다”고 말하는 것만 종료 조건으로 사용하지 말고 테스트, 필수 필드와 비즈니스 규칙을 함께 확인합니다.

### 도구 권한을 최소화하기

읽기 도구와 쓰기 도구를 분리하고, 사용자마다 실제로 허용된 도구만 노출합니다. 도구 인자는 서버에서 다시 검증하고 SQL, 파일 경로와 shell 명령을 모델 출력 그대로 실행하지 않습니다.

### Side effect를 멱등하게 만들기

checkpoint 재개와 재시도 때문에 node가 다시 실행될 수 있습니다. 결제와 메시지 발송에는 idempotency key를 사용하고 실행 완료 여부를 외부 시스템에서도 확인합니다.

### 모델 출력과 도구 결과를 검증하기

구조화 출력에는 스키마 검증을 적용하고, 도구 결과에는 오류 코드와 출처를 포함합니다. 모델이 도구 오류 메시지를 정상 결과로 오해하지 않도록 성공과 실패를 명확히 구분합니다.

### 관찰 가능성과 평가 준비하기

어떤 prompt와 tool이 사용됐고 각 단계에 얼마나 걸렸는지 추적해야 실패를 재현할 수 있습니다. LangSmith는 LangChain 생태계의 tracing·evaluation 도구이지만 필수는 아닙니다. 자체 OpenTelemetry, 로그와 평가 시스템을 사용할 수도 있습니다.

로그에는 prompt, 개인정보와 도구 결과가 들어갈 수 있습니다. 운영 편의를 위해 모든 내용을 무기한 저장하지 말고 마스킹, 접근 통제와 보존 기간을 적용합니다.
{: .notice--warning}

### 테스트를 node 단위로 나누기

- 순수 Python node는 일반 unit test로 검증합니다.
- 모델 node는 고정된 mock 응답과 실제 모델 평가를 나눕니다.
- routing은 경계값마다 올바른 node를 선택하는지 확인합니다.
- tool은 인증, 잘못된 인자, timeout과 재시도를 테스트합니다.
- graph 전체는 대표 시나리오와 실패·재개 시나리오를 검증합니다.

LLM 출력은 확률적이므로 문자열 전체를 snapshot으로 고정하기보다 구조, 필수 사실, 금지 행동과 도구 호출 결과를 평가합니다.

## 흔히 생기는 오해

### “LangChain을 쓰면 모델 성능이 좋아진다”

LangChain 자체가 기반 모델의 지능을 높이지는 않습니다. 적절한 prompt, 검색 자료와 tool을 일관되게 전달해 애플리케이션 수준의 성공률을 높이는 데 도움을 줍니다.

### “LangGraph는 여러 agent를 쓸 때만 필요하다”

단일 모델을 사용해도 승인, 재시도, 상태 저장과 복구가 필요하면 LangGraph가 유용합니다. 반대로 여러 모델이 있어도 호출 순서가 고정돼 있다면 단순 chain으로 충분할 수 있습니다.

### “Graph를 만들면 모든 경로가 결정적이다”

Edge는 결정적이어도 node 안의 LLM 출력은 확률적입니다. 구조화 출력, 규칙 기반 validator와 제한된 retry로 불확실성을 경계 안에 가둬야 합니다.

### “Memory가 있으면 모델이 모든 대화를 기억한다”

Checkpointer는 state를 저장하지만 모델은 현재 호출에 전달된 컨텍스트만 직접 봅니다. 긴 기록은 잘라내거나 요약·검색해야 하며 이 과정에서 정보가 손실될 수 있습니다.

### “Agent가 도구를 호출했으니 결과가 맞다”

올바른 도구를 잘못된 인자로 호출할 수 있고, 도구 자체도 오래되거나 잘못된 데이터를 반환할 수 있습니다. tool call과 결과 모두 검증 대상입니다.

## 어떤 순서로 학습하면 좋은가

1. 모델을 `invoke`하고 Message 구조를 확인합니다.
2. Prompt Template과 구조화 출력을 연결합니다.
3. `@tool` 함수 하나를 모델에 bind해 tool call 메시지를 관찰합니다.
4. `create_agent`로 모델-도구 반복을 실행합니다.
5. `StateGraph`에서 LLM 없는 node와 edge를 먼저 구성합니다.
6. conditional edge와 명확한 종료 조건을 추가합니다.
7. node 하나를 실제 LLM 호출로 교체합니다.
8. checkpointer와 `thread_id`로 상태 재개를 실험합니다.
9. interrupt로 사람 승인 흐름을 만듭니다.
10. 마지막에 streaming, subgraph, 병렬 실행과 운영 저장소를 추가합니다.

이 순서를 따르면 프레임워크의 편의 기능 뒤에서 메시지와 상태가 실제로 어떻게 변하는지 놓치지 않을 수 있습니다.

## 정리

- **LangChain**은 LLM 애플리케이션의 모델, 메시지, prompt, tool, 검색, 구조화 출력과 agent를 공통 방식으로 연결합니다.
- **LangGraph**는 실행을 state, node와 edge로 표현해 분기, 반복, 병렬 처리, 저장, 중단과 재개를 제어합니다.
- LangChain v1의 `create_agent`는 내부적으로 LangGraph를 사용합니다.
- 일반적인 agent는 LangChain으로 빠르게 만들고, 비즈니스 규칙이 있는 복잡한 workflow는 LangGraph로 명시합니다.
- 모든 문제를 agent로 만들기보다 고정 코드, chain, agent와 graph 중 가장 단순한 구조를 선택합니다.
- 운영 품질은 prompt보다 상태 설계, 도구 권한, 종료 조건, 멱등성, 관찰과 평가에서 결정되는 경우가 많습니다.

LangChain은 **무엇을 모델과 연결할지**, LangGraph는 **그 연결을 어떤 순서와 상태로 실행할지**에 초점을 둔다고 기억하면 두 도구의 역할을 구분하기 쉽습니다.

## 참고 자료

- [LangChain overview](https://docs.langchain.com/oss/python/langchain/overview)
- [LangChain v1의 변경 사항](https://docs.langchain.com/oss/python/releases/langchain-v1)
- [LangChain Agents](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain Models](https://docs.langchain.com/oss/python/langchain/models)
- [LangChain Messages](https://docs.langchain.com/oss/python/langchain/messages)
- [LangChain Tools](https://docs.langchain.com/oss/python/langchain/tools)
- [LangChain Structured output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [LangChain Context engineering](https://docs.langchain.com/oss/python/langchain/context-engineering)
- [LangChain Short-term memory](https://docs.langchain.com/oss/python/langchain/short-term-memory)
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph Streaming](https://docs.langchain.com/oss/python/langgraph/streaming)
