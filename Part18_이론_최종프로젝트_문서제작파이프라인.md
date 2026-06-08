# Part 18. 최종 프로젝트 — 문서 제작 파이프라인

> **이 파트의 목표 (졸업 🎓)**
> 18개 파트의 모든 것을 하나로 묶음. 체인·파서(초급) · 도구·RAG·에이전트·메모리(중급) · 그래프·HITL·멀티에이전트·관측·신뢰성(고급)을 **계층적으로 결합한** 완성형 시스템을 설계·구현함. 주제는 지금까지 이어온 **문서 제작** — "조사 → 작성 → 검토 → (반려 시 수정 루프) → 사람 승인 → 발행"이 보장되는 프로덕션급 파이프라인임. 새 개념은 없음. 배운 도구들을 **올바른 자리에** 끼워 맞추는, 종합과 졸업의 시간임.

---

## 18-1. 최종 설계 — 전체 아키텍처

### 개념

만들 것은 **문서 제작 파이프라인**임. 사용자가 주제를 주면, 시스템이 자료를 조사하고, 보고서를 작성하고, 검토해서 통과할 때까지 다듬고, 사람의 최종 승인을 받아 발행함. 이 흐름은 **반드시 순서대로·검토를 거쳐·승인 후 발행**되어야 하므로(Part 11에서 마주한 그 요구), 그래프로 흐름을 보장함.

### 비유

이 시스템은 **출판사**와 같음. 편집장(그래프)이 전체 공정을 관리하고, 자료 조사원·집필자·교정자(전문 에이전트)가 각 단계를 맡으며, 발행 전 발행인의 최종 결재(HITL)를 받음. 그리고 모든 원고는 장부에 기록되고(영속), 무한 교정 루프를 막는 규칙(안전장치)이 있음. 한 사람의 만능 작가가 아니라, **공정과 팀과 결재가 있는 조직**임.

### 원리 — 어떤 구조를 어디에 쓰는가

핵심은 **각 도구를 올바른 자리에** 두는 것임(Part 12의 사다리·계층).

```
[ StateGraph — 전체 흐름 보장 (Part 13) ]
  │
  ├─ research_node : create_agent (Part 9) + RAG 도구 (Part 7·8)  ← 자유 판단
  ├─ write_node    : 체인/모델 호출 (Part 3·4)                    ← 정해진 변환
  ├─ review_node   : 검토 → 조건 분기 (Part 13)                   ← 통과/반려
  │     └─ 반려 시 → write_node로 루프 (Part 13), 횟수 제한 (Part 17)
  ├─ approval_node : interrupt — 사람 승인 (Part 14)              ← 비가역 직전
  └─ publish_node  : 발행
  
  + 체크포인터 (Part 10·17 영속) · recursion_limit (Part 17) · 트레이싱 (Part 16)
```

- **흐름의 뼈대** = StateGraph (보장이 필요하므로)
- **각 노드의 속** = 에이전트·체인·RAG (자리에 맞게)
- **안전·운영** = HITL·영속·안전장치·관측 (프로덕션이므로)

즉 **그래프 안에 에이전트, 에이전트 안에 도구·RAG** — 배운 계층이 그대로 중첩됨.

### 📌 참고
> 설계의 첫 질문은 항상 "흐름을 보장해야 하나, 자유 판단이 좋나?"임. 보장이 필요한 큰 공정은 그래프로, 그 안의 열린 작업(조사 등)은 에이전트로 — 이 판단이 좋은 아키텍처의 출발임.

### 실무 포인트
- 전체를 한 번에 만들지 말고, 뼈대(그래프) → 노드 채우기 → 안전·운영 순으로 쌓음(18-2~4).
- "이 단계는 보장이 필요한가, 자유가 좋은가"를 노드마다 물으며 구조를 고름.

### 3줄 요약
1. 문서 제작 파이프라인 — 조사→작성→검토→승인→발행의 보장된 공정.
2. 그래프(뼈대) 안에 에이전트·체인·RAG(속)를 계층적으로 결합함.
3. 각 도구를 올바른 자리에 두는 것이 설계의 핵심임.

---

## 18-2. 뼈대 — StateGraph로 흐름 보장

### 개념

먼저 **상태와 그래프 뼈대**를 세움(Part 13). 공정의 단계를 노드로, 순서를 엣지로, 검토 결과를 조건 분기로 그림. 이 뼈대가 "반드시 검토를 거치고, 승인 후 발행"을 보장함.

```python
from typing import TypedDict, Annotated
import operator
from langgraph.graph import StateGraph, START, END

class DocState(TypedDict):
    topic: str
    research: str
    draft: str
    review: str
    approved: bool
    attempts: int
    history: Annotated[list, operator.add]

def route_review(state: DocState) -> str:
    if state["review"] == "통과":
        return "approve"
    if state["attempts"] >= 3:       # 안전장치 (Part 17)
        return "give_up"
    return "revise"

builder = StateGraph(DocState)
builder.add_node("research", research_node)
builder.add_node("write", write_node)
builder.add_node("review", review_node)
builder.add_node("approval", approval_node)   # HITL (Part 14)
builder.add_node("publish", publish_node)

builder.add_edge(START, "research")
builder.add_edge("research", "write")
builder.add_edge("write", "review")
builder.add_conditional_edges("review", route_review,
    {"approve": "approval", "revise": "write", "give_up": END})   # 반려→작성 루프
builder.add_conditional_edges("approval", lambda s: "publish" if s["approved"] else END,
    {"publish": "publish", END: END})
builder.add_edge("publish", END)
```

### 비유

뼈대를 세우는 것은 **출판 공정표를 벽에 붙이는 것**과 같음. "조사 → 집필 → 교정 → (반려면 다시 집필) → 발행인 승인 → 발행"이라는 공정이 그림으로 못 박혀, 누구도 단계를 건너뛰거나 순서를 바꿀 수 없음.

### 원리

이 뼈대는 Part 13·14의 결합임 — 조건 분기(통과/반려)·루프(반려→작성)·HITL 분기(승인/거부)가 한 그래프에 모임. **각 노드는 아직 비어 있음**(다음 절에서 채움). 하지만 뼈대만으로도 "흐름이 어떻게 흐를지"가 완전히 결정됨. 이것이 그래프의 힘 — 속을 채우기 전에 **흐름을 먼저 확정**함.

### 📌 참고
> 뼈대를 먼저 그리고 `draw_ascii()`로 확인하면, 구현 전에 "흐름이 맞는지"를 검증할 수 있음. 노드는 일단 빈 함수(`lambda s: {}`)로 두고 흐름부터 맞추는 것이 좋은 순서임.

### 실무 포인트
- 노드를 채우기 전에 흐름(엣지·분기·루프)을 먼저 확정하고 시각화로 검증함.
- 루프(반려→작성)에는 반드시 종료 조건(횟수 제한 + recursion_limit)을 함께 둠.

### 3줄 요약
1. 상태·노드·엣지·분기로 공정의 흐름 뼈대를 먼저 세움.
2. 조건 분기·루프·HITL 분기가 한 그래프에 결합됨(Part 13·14).
3. 속을 채우기 전에 흐름을 먼저 확정·검증함.

---

## 18-3. 속 채우기 — 에이전트·RAG·체인을 노드에

### 개념

이제 빈 노드를 **각 단계에 맞는 도구**로 채움. 자유 판단이 필요한 곳엔 에이전트, 정해진 변환엔 체인, 지식이 필요한 곳엔 RAG(Part 12의 자리 맞추기).

```python
from langchain.agents import create_agent
from langchain_core.tools import tool

# 조사 노드 = 에이전트 + RAG 도구 (자유 판단: 무엇을 얼마나 찾을지)
@tool
def search_reference(query: str) -> str:
    """참고 자료에서 사실·수치를 검색한다."""
    return reference_rag.invoke(query)        # Part 8 RAG

research_agent = create_agent(model=model, tools=[search_reference],
    system_prompt="너는 리서치 전문가야. 주제 관련 사실을 조사해 정리해.")

def research_node(state: DocState) -> dict:
    r = research_agent.invoke({"messages": [("user", f"{state['topic']} 자료 조사")]})
    return {"research": r["messages"][-1].content, "history": ["조사 완료"]}

# 작성 노드 = 체인 (정해진 변환: 자료 → 보고서)
def write_node(state: DocState) -> dict:
    prompt = f"자료: {state['research']}\n검토의견: {state.get('review','')}\n이를 반영해 보고서를 써줘."
    return {"draft": model.invoke(prompt).content, "attempts": state["attempts"] + 1,
            "history": [f"작성(시도 {state['attempts']+1})"]}

# 검토 노드 = 모델 판단 (구조화 출력으로 통과/반려)
def review_node(state: DocState) -> dict:
    ok = any(ch.isdigit() for ch in state["draft"])   # 데모: 근거(수치) 있으면 통과
    return {"review": "통과" if ok else "보강 필요", "history": ["검토"]}
```

### 비유

뼈대(공정표)에 **사람을 배치하는 것**과 같음. 조사 칸엔 능숙한 조사원(에이전트), 집필 칸엔 정해진 양식대로 쓰는 집필자(체인), 교정 칸엔 기준대로 보는 교정자. 각 칸에 그 일에 맞는 전문가를 앉히면 공정이 살아 움직임.

### 원리

여기서 **계층이 실현됨** — 그래프의 한 노드(`research_node`) 안에서 완전한 에이전트(`research_agent`)가 돌고, 그 에이전트 안에서 RAG 도구가 돌고, 그 RAG는 체인임. **그래프 > 에이전트 > RAG/체인**의 중첩이 한 시스템에 살아 있음(Part 12의 핵심 통찰). 각 노드는 "상태를 받아 부분 갱신을 돌려준다"는 Part 13의 규칙만 지키면, 내부에서 무엇을 쓰든 자유임.

### 📌 참고
> 더 키우려면 조사·작성·검토를 각각 전문 에이전트로 만들고 감독자를 두는 멀티에이전트(Part 15)로 확장할 수 있음. 여기서는 그래프 노드에 에이전트를 직접 넣는 형태로, 같은 계층 결합을 더 단순하게 보여줌.

### 실무 포인트
- 노드마다 "자유 판단(에이전트)이냐, 정해진 변환(체인)이냐, 지식 조회(RAG)냐"를 골라 채움.
- 노드 내부가 복잡해지면 그 노드를 다시 작은 그래프로 쪼갤 수도 있음(중첩 그래프).

### 3줄 요약
1. 빈 노드를 자리에 맞는 도구로 채움 — 에이전트·체인·RAG.
2. 그래프>에이전트>RAG/체인의 계층이 한 시스템에 중첩됨.
3. 노드는 "상태→부분 갱신" 규칙만 지키면 내부는 자유.

---

## 18-4. 안전과 운영 — HITL · 신뢰성 · 관측

### 개념

마지막으로 **프로덕션 요소**를 더함 — 발행 전 사람 승인(HITL·Part 14), 영속 상태와 안전장치(Part 17), 그리고 관측(Part 16). 이로써 "잘 도는 프로토타입"이 "안심하고 운영할 시스템"이 됨.

```python
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver   # 운영은 SqliteSaver/PostgresSaver

# 승인 노드 = HITL (비가역 직전)
def approval_node(state: DocState) -> dict:
    decision = interrupt({"question": "이 보고서를 발행할까요?", "draft": state["draft"]})
    return {"approved": decision == "승인", "history": [f"사람 결정: {decision}"]}

def publish_node(state: DocState) -> dict:
    return {"history": ["📤 발행 완료"]}

# 컴파일 — 체크포인터(영속·HITL의 토대)
graph = builder.compile(checkpointer=InMemorySaver())

# 실행 — recursion_limit(안전장치)
config = {"configurable": {"thread_id": "doc-1"}, "recursion_limit": 25}
graph.invoke({"topic": "생성형 AI 도입", "research": "", "draft": "",
              "review": "", "approved": False, "attempts": 0, "history": []}, config=config)
# → approval_node의 interrupt에서 멈춤

graph.invoke(Command(resume="승인"), config=config)   # 사람 승인 → 발행
```

### 비유

이 단계는 **출판사에 결재 라인·기록 보관·안전 규정을 더하는 것**과 같음. 아무리 원고가 좋아도 발행인 결재 없이는 못 나가고(HITL), 모든 원고는 장부에 남으며(영속), 무한 교정으로 마감을 넘기지 않게 규칙이 있음(안전장치). 그리고 공정의 모든 단계가 기록되어, 문제가 생기면 되짚을 수 있음(관측).

### 원리

세 가지가 시스템을 프로덕션급으로 끌어올림.

- **HITL**: `approval_node`의 `interrupt`가 발행(비가역) 직전에 멈춤. 체크포인터가 상태를 얼려, 사람이 며칠 뒤 `Command(resume=...)`로 재개해도 이어짐.
- **신뢰성**: 영속 체크포인터(재시작에도 유지) + `recursion_limit`(무한 루프 차단) + 상태의 `attempts` 제한(이중 안전장치) + 노드 내 예외 처리.
- **관측**: `LANGSMITH_TRACING`을 켜면 전 공정(조사·작성·검토·승인·발행)이 트리로 기록되어, 디버깅·평가·모니터링이 가능(Part 16).

이 셋이 더해지면, 우리가 만든 것은 더 이상 데모가 아니라 **운영 가능한 시스템**임.

### 📌 참고
> 운영 배포 시엔 `InMemorySaver`를 `SqliteSaver`/`PostgresSaver`로 바꾸고(거의 한 줄), 평가셋으로 품질을 측정하며(Part 16), 비용·지연을 최적화함(Part 17). 이 파이프라인이 그 모든 것을 받을 준비가 된 골격임.

### 실무 포인트
- HITL는 비가역 단계(발행)에만 — 조사·작성·검토 같은 가역 단계엔 두지 않음(Part 14 원칙).
- 영속·안전장치·관측은 "나중에"가 아니라 처음부터 골격에 포함하는 것이 안전함.

### 3줄 요약
1. 발행 전 HITL 승인 + 영속 상태 + 안전장치 + 관측을 더함.
2. 체크포인터가 HITL(멈춤·재개)과 영속(재시작 유지)의 공통 토대.
3. 이 셋이 프로토타입을 운영 가능한 시스템으로 끌어올림.

---

## 18-5. 졸업 — 전체 여정 회고

### 개념

축하함. **18개 파트의 여정을 완주**했음. 우리가 만든 문서 제작 파이프라인 하나에, 배운 모든 것이 담겨 있음.

### 우리가 걸어온 길

**🟢 초급 — 기본기 (Part 0~5)**
LangChain 1.0의 그림(에이전트는 LangGraph 위에서 돈다) → 모델 호출 → 프롬프트 템플릿 → LCEL 파이프(`|`) → 출력 파서 → 첫 미니 프로젝트. **정해진 변환(체인)**을 익혔음.

**🟡 중급 — 에이전트 (Part 6~11)**
체인 vs 에이전트(흐름 제어의 주체) → 도구 정의(docstring=명세) → RAG(의미로 검색) → `create_agent`로 결합 → 메모리(상태) → 문서 비서 프로젝트. **자유 판단하는 에이전트**를 익혔고, 그 한계를 마주했음.

**🔴 고급 — 제어와 운영 (Part 12~18)**
왜 LangGraph인가(네 신호) → StateGraph 직접 → HITL → 멀티에이전트 → 관측·평가 → 배포·신뢰성 → 최종 프로젝트. **흐름을 직접 제어하고, 측정하고, 안정적으로 운영**하는 법을 익혔음.

### 관통하는 한 가지 원칙

> **올바른 도구를, 올바른 자리에, 필요한 만큼만.**

체인·에이전트·그래프·멀티에이전트는 위계가 아니라 **도구 상자**임(Part 12·15). 가벼운 것부터 시작해, 천장에 부딪힐 때만 올리고, 계층적으로 중첩함. "최신이라서"가 아니라 "이 문제가 그것을 요구해서" 쓰는 것 — 이 판단력이 커리큘럼이 남긴 가장 중요한 것임.

### 📌 이제 무엇을 할까
- **직접 만들기**: 자신의 도메인 문제에 이 파이프라인 골격을 적용해 봄. 도구·RAG·노드를 자신의 것으로 바꿈.
- **깊이 파기**: 멀티에이전트 패턴(swarm 등), 평가 자동화, 영속·확장 아키텍처를 더 파고듦.
- **공식 문서와 함께**: LangChain/LangGraph는 빠르게 진화함 — 공식 문서로 최신 API를 확인하는 습관을 들임(이 강의도 매 파트 그렇게 했음).

### 실무 포인트
- 배운 것을 자신의 실제 문제 하나에 끝까지 적용해 보는 것이 가장 큰 학습임.
- "왜 이 구조인가"를 네 신호·사다리로 설명할 수 있으면, 어떤 새 도구가 나와도 흔들리지 않음.

### 3줄 요약
1. 18개 파트로 체인→에이전트→그래프·멀티·운영의 전 여정을 완주함.
2. 관통 원칙: 올바른 도구를, 올바른 자리에, 필요한 만큼만.
3. 이제 자신의 문제에 적용하며, 공식 문서와 함께 계속 자람.

---

## 🎓 커리큘럼 졸업

Part 18에서는 배운 전부를 하나로 묶어 운영 가능한 문서 제작 파이프라인을 완성했음:

- 최종 설계 — 그래프 안에 에이전트·RAG, 계층적 결합 (18-1)
- 뼈대 — StateGraph로 흐름 보장 (18-2)
- 속 — 노드를 에이전트·체인·RAG로 채움 (18-3)
- 안전·운영 — HITL·영속·안전장치·관측 (18-4)
- 졸업 회고 — 전 여정과 관통 원칙 (18-5)

**LangChain 에이전트 마스터 커리큘럼(Part 0~18)을 완주했음.** 이제 당신은 단순한 모델 호출부터 프로덕션급 멀티에이전트 시스템까지, **왜·언제·어떻게**를 알고 만들 수 있음. 남은 것은 당신의 문제에 이것을 펼치는 일임.

> 함께 제공되는 **Part 18 실습 노트북**에서는 이 문서 제작 파이프라인을 처음부터 끝까지 직접 구현함 — 상태·뼈대 그래프 → 조사 에이전트(RAG)·작성·검토 노드 채우기 → 검토 통과까지 루프 → 발행 전 HITL 승인(interrupt/resume) → 체크포인터·recursion_limit 안전장치까지. 18개 파트의 조각이 하나의 작동하는 시스템으로 맞춰지는 것을 Gemini로 직접 확인하며 졸업함.
