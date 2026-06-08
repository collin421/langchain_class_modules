# Part 3. LCEL 파이프(|)의 원리

> **이 파트의 목표**
> Part 1에서 모델 블록을, Part 2에서 프롬프트 템플릿 블록을 만들었음. 두 블록 모두 `.invoke()`를 가졌다는 공통점이 있었음. 이 파트에서는 그 공통점이 사실 **약속(Runnable 규격)**이었음을 밝히고, 그 약속 덕분에 블록들을 파이프(`|`) 하나로 줄줄이 이을 수 있음을 배움. 이 `|` 한 글자가 LangChain의 심장인 **LCEL(LangChain Expression Language)**이며, 초급의 정점임.

---

## 3-1. Runnable이란 — 모든 블록이 공유하는 약속

### 개념

지금까지 만든 프롬프트 템플릿과 모델은 둘 다 `.invoke()`로 실행됐음. 이건 우연이 아니라, LangChain의 거의 모든 부품이 **Runnable**이라는 하나의 공통 인터페이스(약속)를 따르기 때문임.

> **Runnable의 약속**: "나는 입력을 받아 출력을 내며, `.invoke()` · `.stream()` · `.batch()`를 가진다."

### 비유

Runnable은 **표준 규격의 전기 콘센트**와 같음. 세상의 모든 전자제품(블록)이 똑같은 모양의 플러그를 쓰기로 약속했기 때문에, 청소기든 선풍기든 같은 콘센트에 꽂아 쓸 수 있음. 만약 제품마다 플러그 모양이 다르면 매번 변환 어댑터가 필요했을 것임.

LangChain에서 프롬프트·모델·파서·심지어 우리가 만든 함수까지 모두 같은 플러그(Runnable)를 쓰므로, 서로 자유롭게 연결됨.

### 원리

`.invoke()`라는 메서드 이름이 모든 블록에서 동일하다는 점이 핵심임. 어떤 블록 A의 출력이 다른 블록 B의 입력으로 들어갈 때, B는 "내가 받은 게 프롬프트인지 모델인지" 신경 쓸 필요가 없음 — 그저 "입력을 받아 `.invoke()`로 처리"하면 됨. 이 통일된 약속이 블록들을 **레고처럼 조립**할 수 있게 만듦.

### 📌 참고
> 앞서 본 `PromptTemplate`, `ChatPromptTemplate`, `ChatGoogleGenerativeAI`가 전부 Runnable임. 다음 파트의 출력 파서, 그리고 우리가 직접 만든 함수(`RunnableLambda`)도 Runnable로 만들 수 있음. "Runnable이면 파이프로 이을 수 있다"가 핵심 규칙임.

### 실무 포인트
- "이 블록을 체인에 끼울 수 있나?"의 답은 거의 항상 "Runnable인가?"임.
- 새 라이브러리·도구를 LangChain에 붙일 때도, Runnable로 감싸면 기존 체인에 자연스럽게 합류함.

### 3줄 요약
1. LangChain의 거의 모든 부품은 Runnable이라는 공통 약속을 따름.
2. 약속의 핵심은 "`.invoke / .stream / .batch`를 가진다"임.
3. 이 통일된 규격 덕분에 블록을 레고처럼 조립할 수 있음.

---

## 3-2. 파이프(|) 연산자 — 블록을 잇다

### 개념

LCEL의 핵심 문법은 단 한 글자, 파이프 `|`임. 왼쪽 블록의 **출력**을 오른쪽 블록의 **입력**으로 흘려보냄.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_google_genai import ChatGoogleGenerativeAI

prompt = ChatPromptTemplate.from_template("{product}의 홍보 문구를 한 줄 써줘")
model = ChatGoogleGenerativeAI(model="gemini-3-flash")

chain = prompt | model        # ← 두 블록을 파이프로 연결

result = chain.invoke({"product": "베이지 니트"})
print(result.content)
```

### 비유

파이프 `|`는 **컨베이어 벨트**임. 한쪽 끝에 재료(입력 딕셔너리)를 올려놓으면, 벨트를 따라 프롬프트 공정 → 모델 공정을 차례로 지나며 가공되고, 반대편 끝에서 완성품(답변)이 나옴.

유닉스/리눅스를 다뤄 봤다면 `cat file | grep "검색어"`의 그 `|`와 정확히 같은 발상임 — "왼쪽 결과를 오른쪽으로 흘려보낸다."

### 원리 — 데이터가 흐르는 방식

`chain.invoke({"product": "베이지 니트"})`를 실행하면 내부에서 이런 일이 순서대로 일어남.

1. **입력** `{"product": "베이지 니트"}`가 `prompt`로 들어감.
2. `prompt`가 빈칸을 채워 **메시지 리스트**를 만들어 출력함.
3. 그 메시지 리스트가 그대로 `model`의 입력으로 흘러 들어감.
4. `model`이 호출되어 **`AIMessage`**를 출력함.
5. 그 `AIMessage`가 체인 전체의 최종 결과가 됨.

즉 `prompt | model`은 사실 "`prompt`의 출력을 `model.invoke()`에 넣는다"를 우아하게 줄여 쓴 것임. 블록을 더 이으면(`prompt | model | parser`) 벨트 공정이 하나 더 늘어날 뿐임.

### 📌 참고
> 파이프로 이은 결과(`chain`)도 **다시 Runnable**임. 그래서 `chain`을 또 다른 더 큰 체인의 한 블록으로 끼워 넣을 수 있음. 이 "조립한 것이 다시 부품이 되는" 성질이 복잡한 파이프라인을 단순하게 유지하는 비결임.

### 실무 포인트
- 보통 체인은 `프롬프트 | 모델 | 출력파서` 3단이 기본 형태임(파서는 Part 4).
- 체인을 만들 때와 실행할 때가 분리됨 — `chain = ...`로 조립해 두고, 필요할 때 `.invoke()`로 돌림.

### 3줄 요약
1. LCEL의 핵심은 파이프 `|` — 왼쪽 출력을 오른쪽 입력으로 흘려보냄.
2. `prompt | model`은 "prompt 출력을 model에 넣는다"를 줄여 쓴 것임.
3. 파이프로 이은 체인도 다시 Runnable이라 더 큰 체인에 끼울 수 있음.

---

## 3-3. invoke · stream · batch가 체인 전체에 적용된다

### 개념

가장 강력한 사실 하나 — **체인 전체가 다시 Runnable이므로**, 체인에도 `.invoke()` · `.stream()` · `.batch()`를 그대로 쓸 수 있음. 개별 블록에 쓰던 그 메서드들이 조립된 파이프라인 전체에도 똑같이 작동함.

```python
chain = prompt | model

chain.invoke({"product": "니트"})              # 한 번에
chain.batch([{"product": "니트"}, {"product": "코트"}])   # 여러 개 한꺼번에
for chunk in chain.stream({"product": "니트"}):          # 실시간 조각
    print(chunk.content, end="")
```

### 비유

`.stream()`을 체인에 걸면, 컨베이어 벨트의 **마지막 공정에서 완성품이 조각조각** 흘러나옴. 모델이 글자를 생성하는 족족 벨트 끝으로 전달되므로, 사용자는 답을 기다리지 않고 타이핑되듯 받아봄.

### 원리

`.stream()`을 체인에 호출하면, LangChain은 스트리밍을 지원하는 마지막 블록(보통 모델)까지 자동으로 그 요청을 전달함. 앞 단계(프롬프트)는 한 번에 처리되고, 모델이 내보내는 조각들이 그 뒤 블록(파서 등)을 거쳐 실시간으로 흘러나옴. 우리가 "어디서 스트리밍이 시작되는지"를 직접 챙길 필요가 없음 — Runnable 규격이 알아서 전파함.

### 📌 참고
> 비동기 버전(`.ainvoke`, `.astream`, `.abatch`)도 체인 전체에 동일하게 적용됨. 웹 서버에서 동시에 많은 사용자를 처리할 때 씀. 지금은 "개별 블록에 쓰던 실행 메서드를 체인 통째로도 쓸 수 있다"만 확실히 잡으면 됨.

### 실무 포인트
- 챗봇 응답은 `chain.stream(...)`으로 받아 화면에 실시간 출력하는 것이 표준임.
- 상품 수백 개에 같은 체인을 돌릴 땐 `chain.batch([...])`가 반복문보다 빠름.

### 3줄 요약
1. 체인 전체가 Runnable이라 `.invoke / .stream / .batch`를 그대로 씀.
2. `.stream()`은 마지막(모델)부터 조각을 실시간으로 흘려보냄.
3. 챗봇은 stream, 대량 처리는 batch가 정석임.

---

## 3-4. RunnablePassthrough · RunnableParallel — 흐름을 다루기

### 개념

체인은 항상 한 줄로만 흐르지 않음. 때로는 **입력을 그대로 다음으로 넘기거나**, **여러 갈래로 동시에 처리**해야 함. 이를 위한 두 도구가 있음.

| 도구 | 역할 |
|---|---|
| `RunnablePassthrough` | 받은 입력을 **가공 없이 그대로** 통과시킴 |
| `RunnableParallel` | 여러 블록을 **동시에(병렬)** 실행해 결과를 딕셔너리로 모음 |

```python
from langchain_core.runnables import RunnablePassthrough, RunnableParallel

# 입력을 두 갈래로: 하나는 그대로, 하나는 길이 계산
parallel = RunnableParallel(
    original=RunnablePassthrough(),          # 입력 그대로
    length=lambda x: len(x),                 # 입력의 길이
)
print(parallel.invoke("베이지 니트"))
# → {'original': '베이지 니트', 'length': 6}
```

### 비유

- `RunnablePassthrough`는 **복사기의 "원본 그대로 통과" 버튼**임. 손대지 않고 다음 공정으로 넘김.
- `RunnableParallel`은 **여러 작업자에게 같은 재료를 동시에 나눠주는 분배기**임. 각자 자기 일을 하고, 결과를 한 바구니(딕셔너리)에 모아 돌려줌.

### 원리

`RunnableParallel`은 같은 입력을 **여러 블록에 동시에** 흘려보내고, 각 블록의 출력을 지정한 키(`original`, `length`)에 담아 하나의 딕셔너리로 합침. 병렬이므로 순차 실행보다 빠름. `RunnablePassthrough`는 "이 갈래에서는 입력을 변형하지 말고 그대로 두라"는 뜻으로, 특히 RAG(Part 8)에서 "질문은 그대로 두고, 동시에 그 질문으로 문서도 검색"하는 패턴에 핵심으로 쓰임.

### 📌 참고
> `RunnablePassthrough.assign(...)`을 쓰면 "기존 입력은 유지하면서 새 키를 추가"할 수 있음. RAG 체인에서 "원래 질문 + 검색된 문서"를 함께 다음으로 넘길 때 자주 등장함. 지금은 "그대로 통과 / 병렬 분배" 두 개념만 잡으면 됨.

### 실무 포인트
- RAG 체인의 전형: `{"context": retriever, "question": RunnablePassthrough()} | prompt | model`. 질문은 그대로 두고 동시에 문서를 검색해 둘 다 프롬프트에 넣음(Part 8에서 본격적으로 다룸).
- 한 입력에서 여러 정보를 동시에 뽑아야 할 때 `RunnableParallel`을 떠올리면 됨.

### 3줄 요약
1. `RunnablePassthrough`는 입력을 가공 없이 그대로 통과시킴.
2. `RunnableParallel`은 여러 블록을 병렬 실행해 딕셔너리로 모음.
3. 이 둘은 RAG 체인(질문 그대로 + 동시에 문서 검색)의 핵심 패턴임.

---

## 3-5. RunnableLambda — 내 함수를 체인에 끼우기

### 개념

체인 중간에 **내가 만든 평범한 파이썬 함수**를 끼워 넣고 싶을 때가 있음. 예를 들어 "모델 출력에서 앞뒤 공백을 제거"하거나 "텍스트를 대문자로" 같은 간단한 가공임. `RunnableLambda`는 일반 함수를 Runnable로 감싸 체인에 합류시킴.

```python
from langchain_core.runnables import RunnableLambda

def shout(text: str) -> str:
    return text.upper() + "!!!"

chain = RunnableLambda(shout)
print(chain.invoke("hello"))   # → HELLO!!!
```

파이프 안에서 함수가 자동으로 Runnable로 변환되기도 함.

```python
chain = prompt | model | (lambda msg: msg.content.strip())
# 모델의 AIMessage에서 본문만 꺼내 공백 제거
```

### 비유

`RunnableLambda`는 **표준 플러그가 안 달린 옛날 제품에 끼우는 변환 어댑터**임. 내 함수는 원래 LangChain 규격(Runnable)이 아니지만, 어댑터로 감싸면 콘센트(파이프)에 꽂을 수 있게 됨.

### 원리

`RunnableLambda(함수)`는 그 함수에 `.invoke()` 등의 Runnable 메서드를 입혀, 체인의 한 블록처럼 행동하게 만듦. 파이프 `|`의 오른쪽이나 왼쪽에 그냥 함수(또는 람다)를 쓰면 LangChain이 이를 자동으로 `RunnableLambda`로 감싸 줌. 덕분에 "모델 출력을 살짝 후처리"하는 코드를 체인 안에 자연스럽게 녹일 수 있음.

### 📌 참고
> 다음 파트에서 배울 출력 파서(`StrOutputParser` 등)도 사실 "모델 출력을 가공하는 Runnable"의 잘 만들어진 버전임. 간단한 가공은 `RunnableLambda`로, 정형화된 파싱은 전용 파서로 한다고 보면 됨.

### 실무 포인트
- 모델 출력에서 `.content`만 꺼내거나, 공백·따옴표를 정리하는 등의 잔손질에 자주 씀.
- 다만 함수가 복잡해지면 별도 함수로 빼서 `RunnableLambda(내함수)`로 명시하는 편이 가독성에 좋음.

### 3줄 요약
1. `RunnableLambda`는 내 일반 함수를 Runnable로 감싸 체인에 끼움.
2. 파이프 안의 함수·람다는 자동으로 RunnableLambda로 변환됨.
3. 모델 출력의 간단한 후처리에 유용하며, 정형 파싱은 전용 파서를 씀.

---

## Part 3 마무리 — 다음으로

Part 3에서는 LangChain의 심장인 LCEL을 손에 쥐었음. 정리하면:

- 모든 블록이 공유하는 약속 **Runnable** — `.invoke / .stream / .batch` (3-1)
- 핵심 문법 파이프 `|` — 왼쪽 출력을 오른쪽 입력으로, 컨베이어 벨트처럼 (3-2)
- 체인 전체도 Runnable이라 실행 메서드가 통째로 적용됨 (3-3)
- `RunnablePassthrough`(그대로 통과) · `RunnableParallel`(병렬 분배) (3-4)
- `RunnableLambda`(내 함수를 체인에 끼우기) (3-5)

이제 우리는 `프롬프트 | 모델`까지 잇는 법을 알지만, 모델의 출력은 아직 **`AIMessage` 객체**임. 깔끔한 문자열이나 구조화된 데이터(JSON 등)로 받으려면 마지막 공정 하나가 더 필요함. **Part 4**에서는 그 마지막 블록인 **출력 파서(Output Parser)**를 배움 — `프롬프트 | 모델 | 파서`의 완성된 3단 체인이 비로소 모습을 드러냄.

> 함께 제공되는 **Part 3 실습 노트북**에서는 `프롬프트 | 모델` 체인을 직접 만들어 `.invoke / .stream / .batch`로 돌려보고, `RunnableParallel`로 한 입력에서 여러 결과를 동시에 뽑으며, `RunnableLambda`로 모델 출력을 후처리하는 실습을 Gemini로 진행함.
