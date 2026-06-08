# Part 4. 출력 파서 (Output Parser)

> **이 파트의 목표**
> Part 3에서 `프롬프트 | 모델`까지 이었지만, 모델의 출력은 아직 **`AIMessage` 객체**임. 사람이 읽을 깔끔한 문자열, 혹은 프로그램이 쓸 구조화된 데이터(JSON, 객체)로 받으려면 마지막 공정 하나가 더 필요함. 이 파트에서 그 마지막 블록인 **출력 파서**를 배우면, 드디어 `프롬프트 | 모델 | 파서`라는 LangChain의 완성된 3단 체인이 모습을 드러냄.

---

## 4-1. 왜 파서인가 — 출력은 가공이 필요하다

### 개념

지금까지 모델을 호출하면 답이 `AIMessage` 객체로 왔고, 본문을 보려면 매번 `.content`를 꺼내야 했음. 게다가 "결과를 표로 정리해줘"라고 해도 모델은 결국 **글자(텍스트)**만 뱉음 — 프로그램이 그 텍스트에서 값을 자동으로 꺼내 쓰려면 누군가 그것을 **파싱(해석)**해야 함.

**출력 파서(Output Parser)**는 모델의 텍스트 출력을 받아, 우리가 원하는 형태(깔끔한 문자열, 리스트, JSON, 객체 등)로 **변환해주는 마지막 블록**임.

### 비유

모델의 출력은 **손글씨로 적힌 주문서**와 같음. 내용은 다 있지만, 그대로는 프로그램이 쓰기 어려움. 출력 파서는 그 손글씨를 받아 **정해진 양식의 전산 입력 폼**으로 옮겨 적는 사무원임. "수량 칸엔 숫자만, 날짜 칸엔 날짜 형식으로" 정리해 넘겨줌.

### 원리

출력 파서 역시 Part 3에서 배운 **Runnable**임 — `.invoke()`를 가짐. 그래서 파이프 맨 끝에 그냥 이어 붙이면 됨.

```python
chain = prompt | model | parser
#                         └── 모델 출력(AIMessage)을 받아 원하는 형태로 변환
```

데이터 흐름: 프롬프트가 메시지를 만들고 → 모델이 `AIMessage`를 내고 → 파서가 그걸 받아 최종 형태(문자열·JSON·객체)로 바꿔 체인의 결과로 내보냄.

### 📌 참고
> 파서는 크게 둘로 나뉨. 단순히 텍스트를 **깔끔히 다듬는** 파서(`StrOutputParser`)와, 텍스트를 **구조화된 데이터로 해석하는** 파서(`JsonOutputParser`, Pydantic 기반 등)임. 후자가 실무에서 모델을 "데이터 추출기"로 쓸 때 핵심이 됨.

### 실무 포인트
- 사람에게 보여줄 답이면 `StrOutputParser`로 충분함.
- 모델의 답을 **다른 코드가 받아 처리**해야 하면(DB 저장, 분기 판단 등) 구조화 파서가 필수임. "텍스트를 눈으로 읽고 손으로 옮기는" 일을 없애 줌.

### 3줄 요약
1. 모델 출력은 `AIMessage`이고, 결과는 결국 텍스트라 가공이 필요함.
2. 출력 파서는 그 텍스트를 원하는 형태로 바꾸는 마지막 블록임.
3. 파서도 Runnable이라 파이프 끝에 `| parser`로 이어 붙임.

---

## 4-2. StrOutputParser — 가장 기본

### 개념

`StrOutputParser`는 모델의 `AIMessage`에서 **본문 문자열만 깔끔히 꺼내** 주는 가장 단순한 파서임. Part 3에서 `(lambda m: m.content)`로 하던 일을 정식 블록으로 만든 것임.

```python
from langchain_core.output_parsers import StrOutputParser

chain = prompt | model | StrOutputParser()

result = chain.invoke({"product": "베이지 니트"})
print(type(result))   # <class 'str'> — 이제 그냥 문자열!
print(result)         # 포근함을 입다 — 베이지 니트
```

### 비유

`StrOutputParser`는 **택배 상자에서 알맹이만 꺼내 건네주는 것**과 같음. `AIMessage`라는 포장(상자)을 벗기고 안의 내용물(문자열)만 손에 쥐여 줌.

### 원리

`StrOutputParser`는 입력으로 받은 `AIMessage`의 `.content`를 추출해 순수 `str`로 돌려줌. 스트리밍(`.stream()`)에서도 조각마다 문자열만 흘려보내므로, 실시간 출력 코드가 한결 깔끔해짐(`chunk.content` 대신 `chunk`가 바로 문자열).

### 📌 참고
> `StrOutputParser`를 붙인 체인은 결과가 `str`이므로, 그 뒤에 또 다른 텍스트 처리 블록(예: `RunnableLambda`로 해시태그 변환)을 자연스럽게 이을 수 있음. "문자열로 받아두면 그 다음 가공이 쉬워진다"가 핵심 효용임.

### 실무 포인트
- 사람이 읽을 답을 만드는 거의 모든 체인의 기본 마무리임 — `prompt | model | StrOutputParser()`.
- 챗봇 스트리밍에서 특히 유용함. `for chunk in chain.stream(...)`의 `chunk`가 바로 출력할 문자열이 됨.

### 3줄 요약
1. `StrOutputParser`는 `AIMessage`에서 본문 문자열만 깔끔히 추출함.
2. 결과 타입이 `str`이라 이후 텍스트 가공이 쉬워짐.
3. 사람에게 보여줄 답을 만드는 체인의 기본 마무리임.

---

## 4-3. JsonOutputParser — 구조화된 데이터로 받기

### 개념

모델에게 "상품명, 가격대, 추천 코디 3가지를 알려줘"라고 하면, 그 결과를 **프로그램이 바로 쓸 수 있는 JSON(딕셔너리)**으로 받고 싶을 때가 많음. `JsonOutputParser`는 모델의 텍스트 출력을 파싱해 파이썬 딕셔너리/리스트로 변환해 줌.

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()

prompt = ChatPromptTemplate.from_template(
    "다음 상품 정보를 JSON으로 정리해줘. 키는 name, price_range, codi.\n상품: {product}\n{format_instructions}"
).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
result = chain.invoke({"product": "베이지 니트"})

print(type(result))        # <class 'dict'>
print(result["codi"])      # 키로 바로 접근 가능
```

### 비유

`JsonOutputParser`는 **자유 서술형 답안을 표 양식으로 옮겨 정리하는 채점자**임. "이 상품은 베이지 니트이고 5만원대이며..."라는 줄글을, `{name: ..., price_range: ..., codi: [...]}`라는 깔끔한 칸으로 분류해 넣어줌.

### 원리

핵심은 두 가지가 맞물려 돌아간다는 점임.

1. **파서가 모델에게 형식을 알려줌**: `parser.get_format_instructions()`가 "이런 JSON 형식으로 답하라"는 지시문을 만들어 줌. 이걸 프롬프트에 끼워 넣음(`format_instructions` 변수).
2. **파서가 모델 출력을 해석함**: 모델이 그 지시대로 JSON 텍스트를 내면, 파서가 이를 파이썬 딕셔너리로 변환함.

즉 파서는 **"앞에서는 모델에게 형식을 주문하고, 뒤에서는 그 결과를 해석하는"** 양방향 역할을 함. 이 구조가 다음 절들의 공통 뼈대임.

### 📌 참고
> 모델이 가끔 JSON 형식을 어겨(앞에 설명을 붙이거나 따옴표를 빠뜨려) 파싱이 실패할 수 있음. 이를 대비한 `OutputFixingParser`(실패 시 모델로 고치기) 같은 도구도 있지만, 더 안정적인 방법은 다음 절의 `with_structured_output`임.

### 실무 포인트
- `format_instructions`를 프롬프트에 꼭 넣어야 모델이 형식을 맞춤. 빠뜨리면 파싱 실패가 잦음.
- 추출 작업은 `temperature=0`으로 두면 형식이 더 안정적임(Part 1).

### 3줄 요약
1. `JsonOutputParser`는 모델 텍스트를 파이썬 딕셔너리/리스트로 변환함.
2. `get_format_instructions()`로 모델에게 형식을 지시하고, 그 출력을 파서가 해석함(양방향).
3. 형식 위반으로 실패할 수 있어 `temperature=0`과 다음 절 기법이 권장됨.

---

## 4-4. Pydantic 구조화 출력 — 타입까지 보장하기

### 개념

JSON으로 받아도 "가격은 숫자여야 하는데 문자열로 왔다" 같은 문제가 생길 수 있음. **Pydantic 모델**로 출력의 **구조와 타입을 미리 정의**해 두면, 모델 출력이 그 틀에 맞는지 검증까지 됨. 그리고 LangChain 1.0의 권장 방식인 **`with_structured_output`**을 쓰면 이 과정이 매우 깔끔해짐.

```python
from pydantic import BaseModel, Field

# 1) 원하는 출력 구조를 클래스로 정의
class Product(BaseModel):
    name: str = Field(description="상품 이름")
    price_range: str = Field(description="예상 가격대")
    codi: list[str] = Field(description="추천 코디 3가지")

# 2) 모델에 구조를 입힘 — 파이프 없이 모델 자체가 구조화 출력을 함
structured_model = model.with_structured_output(Product)

result = structured_model.invoke("베이지 니트를 정리해줘")
print(type(result))     # <class 'Product'>
print(result.name)      # 점(.)으로 접근, 타입 보장됨
print(result.codi)      # list[str]
```

### 비유

Pydantic 모델은 **입국 신고서 양식**과 같음. "이름은 글자, 나이는 숫자, 입국일은 날짜" 칸이 미리 정해져 있어, 잘못된 형식으로 적으면 곧바로 반려됨. 모델의 답이 이 양식을 통과해야만 결과로 인정되므로, 뒤 코드가 타입 걱정 없이 값을 씀.

### 원리

`with_structured_output(Product)`는 Pydantic 클래스의 필드 이름·타입·설명(`Field(description=...)`)을 모델에게 **구조 명세로 전달**하고, 모델이 그 구조에 맞춰 답하도록 강제함. 그리고 결과를 **`Product` 객체로 바로** 돌려줌. JSON 텍스트를 파싱하는 중간 단계 없이, 타입이 검증된 객체가 손에 들어옴 — 그래서 별도 `format_instructions`나 `JsonOutputParser`를 붙일 필요가 없음.

### 📌 참고
> `with_structured_output`은 모델 쪽 기능(구조화 출력)을 직접 쓰는 방식이라, 텍스트를 사후 파싱하는 `PydanticOutputParser`보다 안정적임. `PydanticOutputParser`는 구조화 출력을 지원하지 않는 환경에서 쓰는 대안으로 이해하면 됨. 핵심 개념은 "출력 구조를 클래스로 정의한다"로 동일함.

### 실무 포인트
- 모델 출력을 DB에 저장하거나, 값으로 분기 판단을 하거나, 다른 API에 넘길 때 거의 필수임 — 타입이 보장돼야 안전함.
- `Field(description=...)`을 성의 있게 적을수록 모델이 각 칸을 정확히 채움. 설명이 곧 모델에게 주는 지시임.

### 3줄 요약
1. Pydantic 모델로 출력의 구조·타입을 클래스로 정의함.
2. `with_structured_output(클래스)`가 구조를 강제하고 검증된 객체를 바로 돌려줌(1.0 권장).
3. DB 저장·분기·API 연동 등 출력을 코드가 받아 쓸 때 핵심임.

---

## 4-5. format_instructions — 파서가 모델에게 형식을 주문하는 원리

### 개념

4-3에서 슬쩍 등장한 `get_format_instructions()`를 정면으로 다룸. 이것은 파서가 **"내가 해석할 수 있게, 모델에게 이런 형식으로 답하라"고 적어 주는 설명서**임. 구조화 파서가 잘 작동하려면, 모델이 먼저 그 형식대로 답해야 하기 때문임.

```python
parser = JsonOutputParser(pydantic_object=Product)
print(parser.get_format_instructions())
# → "다음 JSON 스키마에 맞춰 출력하라: {...필드와 타입 설명...}"
```

### 비유

`format_instructions`는 **택배 송장 양식 안내문**임. 사무원(파서)이 손글씨를 옮겨 적기 쉽도록, 미리 발송인에게 "주소는 이 칸에, 우편번호는 숫자 5자리로 적어 주세요"라고 안내문을 보내는 것과 같음. 안내문대로 적어 보내면(모델), 사무원은 막힘없이 전산 입력(파싱)을 함.

### 원리

구조화 파서는 자신이 해석할 수 있는 형식을 **스스로 알고 있음**. `get_format_instructions()`는 그 형식을 자연어 + 스키마로 풀어낸 텍스트를 만들어 줌. 이 텍스트를 프롬프트에 `{format_instructions}` 변수로 끼워 넣으면(보통 `partial`로 고정), 모델은 그 안내에 따라 형식을 맞춰 답함. 그러면 파서가 그 출력을 정확히 해석함 — **"형식을 주문하는 앞단"과 "형식을 해석하는 뒷단"이 한 짝**으로 맞물리는 것임.

### 📌 참고
> `with_structured_output`을 쓰면 이 `format_instructions` 과정을 우리가 직접 챙기지 않아도 됨(모델 기능으로 내부 처리). 그래서 4-4가 더 간결한 것임. 다만 `format_instructions`의 원리를 알면 "왜 구조화 출력이 되는가"를 이해하게 되어, 형식이 깨질 때 디버깅이 쉬워짐.

### 실무 포인트
- 구조화 파서를 직접 쓸 땐 `format_instructions`를 프롬프트에 넣는 것을 잊지 말 것. 누락이 형식 깨짐의 1순위 원인임.
- 가능하면 `with_structured_output`을 우선 쓰고, 그게 어려운 환경에서만 `JsonOutputParser` + `format_instructions` 조합을 씀.

### 3줄 요약
1. `format_instructions`는 파서가 모델에게 "이 형식으로 답하라"고 주는 설명서임.
2. 프롬프트에 끼워 넣으면 모델이 형식을 맞추고, 파서가 그 출력을 해석함(한 짝).
3. `with_structured_output`은 이 과정을 자동 처리해 더 간결함.

---

## Part 4 마무리 — 초급의 완성

Part 4에서는 체인의 마지막 블록을 손에 쥐었음. 정리하면:

- 모델 출력(`AIMessage`)은 가공이 필요하고, 파서가 그 마지막 블록임 (4-1)
- `StrOutputParser` — 본문 문자열만 깔끔히, 사람용 답의 기본 (4-2)
- `JsonOutputParser` — 텍스트를 딕셔너리/리스트로, 코드용 데이터 (4-3)
- Pydantic + `with_structured_output` — 구조·타입까지 보장된 객체 (4-4)
- `format_instructions` — 파서가 모델에게 형식을 주문하는 원리 (4-5)

이제 우리는 LangChain 초급의 모든 부품을 갖췄음 — **모델(1) · 프롬프트(2) · 파이프/LCEL(3) · 파서(4)**. 이 넷을 조합한 `프롬프트 | 모델 | 파서`가 거의 모든 단순 LLM 앱의 뼈대임.

**Part 5**에서는 지금까지 배운 모든 것을 합쳐, 실제로 작동하는 **첫 미니 프로젝트**(예: 상품 정보를 받아 구조화된 마케팅 카드를 생성하는 체인)를 처음부터 끝까지 만들어 봄. 초급의 졸업 작품이자, 중급의 "체인 vs 에이전트"로 넘어가기 직전의 마지막 디딤돌임.

> 함께 제공되는 **Part 4 실습 노트북**에서는 `StrOutputParser`로 깔끔한 체인을 만들고, `JsonOutputParser`로 상품 정보를 딕셔너리로 받으며, Pydantic + `with_structured_output`으로 타입이 보장된 객체를 뽑아내는 실습을 Gemini로 진행함.
