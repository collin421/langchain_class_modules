# Part 2. 프롬프트 템플릿 (PromptTemplate)

> **이 파트의 목표**
> Part 1에서는 메시지를 매번 손으로 적어 모델에 넣었음. 하지만 실무에서는 **"형식은 똑같고 내용만 바뀌는"** 질문을 수없이 반복함. 이 파트에서는 빈칸이 뚫린 **틀(템플릿)**을 한 번 만들어 두고, 빈칸만 갈아 끼워 질문을 찍어내는 법을 익힘. 이 "틀 + 변수" 구조가 곧 다음 파트의 LCEL 파이프(`|`)로 이어지는 결정적 다리임.

---

## 2-1. 왜 템플릿인가 — 문자열 직접 조립의 한계

### 개념

같은 작업을 여러 입력에 반복할 때, 프롬프트 문자열을 직접 이어 붙이면 금세 지저분해짐.

```python
# 직접 조립 방식 — 입력이 바뀔 때마다 문자열을 다시 만듦
product = "베이지 니트"
prompt = "다음 상품의 홍보 문구를 한 줄로 써줘: " + product
```

상품이 100개라면 이 `+` 조립을 100번 해야 하고, "한 줄로"를 "두 줄로"로 바꾸려면 모든 곳을 손봐야 함. 게다가 변수를 빠뜨리거나 따옴표를 잘못 닫는 실수가 잦음.

### 비유

문자열 직접 조립은 **편지를 매번 손으로 처음부터 쓰는 것**과 같음. 100명에게 같은 안내문을 보낸다면, 이름만 다른 편지를 100번 새로 쓰는 셈임.

**프롬프트 템플릿**은 **빈칸이 뚫린 양식지**임. "○○○ 고객님께"라고 미리 인쇄해 두고, 이름만 채워 넣으면 됨. 양식은 한 번만 만들고, 빈칸만 바꿔 끼움.

### 원리

LangChain의 프롬프트 템플릿은 문자열 안에 **중괄호 변수** `{변수명}`를 두고, 나중에 그 변수에 값을 채워 완성된 프롬프트를 만들어내는 블록임. 그리고 이 템플릿 역시 Part 0에서 말한 **Runnable 규격**을 따름 — 즉 `.invoke()`를 가짐. 그래서 나중에 모델과 파이프(`|`)로 이을 수 있음.

### 📌 참고
> 프롬프트 템플릿은 두 종류가 있음. 단일 문자열을 다루는 **`PromptTemplate`**, 그리고 System/Human 같은 역할이 붙은 메시지를 다루는 **`ChatPromptTemplate`**임. 채팅 모델(Part 1의 ChatModel)을 쓸 때는 후자가 기본임.

### 실무 포인트
- 프롬프트를 코드 곳곳에 흩어 놓지 말고, 템플릿으로 **한 곳에 모아** 관리함. 수정·버전 관리가 쉬워짐.
- 리리마켓처럼 상품 수백 개에 같은 형식의 문구를 생성한다면, 템플릿 하나 + 반복문이 정석임.

### 3줄 요약
1. 문자열 직접 조립은 반복·수정·실수에 약함.
2. 프롬프트 템플릿은 빈칸이 뚫린 양식지로, 변수만 갈아 끼움.
3. 템플릿도 Runnable 규격이라 나중에 모델과 파이프로 이어짐.

---

## 2-2. PromptTemplate — 단일 문자열 템플릿

### 개념

`PromptTemplate`은 **하나의 문자열** 안에 변수를 둔 가장 기본 템플릿임. `from_template()`으로 만들고, 값을 넣어 완성함.

```python
from langchain_core.prompts import PromptTemplate

template = PromptTemplate.from_template(
    "다음 상품의 홍보 문구를 {tone} 톤으로 한 줄 써줘: {product}"
)

# 빈칸 채우기
prompt = template.invoke({"product": "베이지 니트", "tone": "감성적인"})
print(prompt.text)
# → 다음 상품의 홍보 문구를 감성적인 톤으로 한 줄 써줘: 베이지 니트
```

### 문법 핵심

| 요소 | 의미 |
|---|---|
| `PromptTemplate.from_template("...")` | 문자열에서 템플릿을 만듦. `{변수}`를 자동 인식함 |
| `{product}`, `{tone}` | 채워 넣을 빈칸(변수) |
| `.invoke({...})` | 딕셔너리로 변수에 값을 채워 완성함 |
| `.format(**kwargs)` | `.format(product="...", tone="...")` 형태로도 채울 수 있음 |

### 비유

`from_template`은 **도장 새기기**임. "______ 님 환영합니다"라는 도장을 한 번 새겨 두면(`from_template`), 종이마다 이름만 찍어(`invoke`) 무한히 찍어낼 수 있음.

### 원리

`from_template`이 문자열을 받으면, 내부에서 `{ }`로 감싼 부분을 찾아 **변수 목록(`input_variables`)**으로 자동 등록함. 그 뒤 `.invoke()`에 딕셔너리를 넘기면, 등록된 변수 자리에 값을 끼워 넣어 완성된 프롬프트를 돌려줌.

### 📌 참고
> 문자열에 실제 중괄호 문자 `{`나 `}`를 그대로 넣고 싶으면 `{{`, `}}`처럼 두 번 씀(변수로 오인되지 않게 하기 위함). 예: JSON 예시를 프롬프트에 넣을 때 자주 마주침.

### 실무 포인트
- `PromptTemplate`은 채팅 모델보다는 단순 텍스트 완성 모델·간단한 체인에서 씀. 채팅 모델을 쓸 땐 다음 절의 `ChatPromptTemplate`이 기본임.
- 변수명은 의미가 드러나게(`product`, `tone`) 지어야 나중에 헷갈리지 않음.

### 3줄 요약
1. `PromptTemplate`은 단일 문자열에 `{변수}`를 둔 기본 템플릿임.
2. `from_template`으로 만들고 `.invoke({...})`로 빈칸을 채움.
3. 변수는 자동으로 `input_variables`에 등록됨.

---

## 2-3. ChatPromptTemplate — 메시지 템플릿

### 개념

채팅 모델은 Part 1에서 봤듯 **역할이 붙은 메시지 리스트**로 대화함. `ChatPromptTemplate`은 그 메시지들 각각을 템플릿으로 만들어 둠. System/Human 역할을 그대로 살리면서 변수를 끼워 넣을 수 있음.

```python
from langchain_core.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_messages([
    ("system", "너는 리리마켓의 {tone} 쇼핑 도우미야."),
    ("human", "{product}에 어울리는 코디를 추천해줘."),
])

prompt = template.invoke({"tone": "친절한", "product": "베이지 니트"})
print(prompt.messages)
# → [SystemMessage('너는 리리마켓의 친절한 쇼핑 도우미야.'),
#    HumanMessage('베이지 니트에 어울리는 코디를 추천해줘.')]
```

### 문법 핵심

| 요소 | 의미 |
|---|---|
| `ChatPromptTemplate.from_messages([...])` | 메시지 템플릿 리스트로 만듦 |
| `("system", "...")` | 역할 + 템플릿 문자열의 **튜플**. `"system"`, `"human"`, `"ai"` 사용 |
| `{변수}` | 각 메시지 안에서도 동일하게 빈칸으로 작동함 |
| `.invoke({...})` | 모든 메시지의 빈칸을 한 번에 채워 메시지 리스트를 완성함 |

### 비유

`ChatPromptTemplate`은 **연극 대본 양식**임. "집사 역(System): ○○한 말투로 말하라"와 "손님 역(Human): ○○에 대해 묻는다"가 각각 빈칸과 함께 적혀 있고, 공연마다 빈칸만 채워 대본을 완성함.

### 원리

`from_messages`에 넘긴 `("역할", "템플릿")` 튜플 각각이 내부에서 해당 역할의 메시지 템플릿으로 변환됨. `.invoke()` 시 모든 메시지의 변수가 동시에 채워지고, 결과는 Part 1에서 본 그 메시지 리스트(`SystemMessage`, `HumanMessage`...)가 됨. 그래서 이 결과를 그대로 ChatModel에 넘길 수 있음.

### 📌 참고
> 대화 기록(이전 메시지들)을 통째로 끼워 넣을 자리가 필요하면 `MessagesPlaceholder`를 씀. 멀티턴·메모리(Part 10)에서 다시 만남. 지금은 "역할별 템플릿을 한 묶음으로 만든다"만 이해하면 충분함.

### 실무 포인트
- 채팅 모델을 쓰는 거의 모든 실무 코드는 `ChatPromptTemplate`을 기본으로 함.
- SystemMessage 자리에 브랜드 톤·규칙·금지사항을 변수로 빼 두면, 같은 양식으로 여러 브랜드·말투를 운용할 수 있음.

### 3줄 요약
1. `ChatPromptTemplate`은 역할이 붙은 메시지들을 템플릿으로 묶음.
2. `("system", "...")` 같은 튜플로 역할과 템플릿을 함께 지정함.
3. `.invoke()` 결과는 ChatModel에 그대로 넣을 수 있는 메시지 리스트임.

---

## 2-4. 변수 주입의 원리 — input_variables와 partial

### 개념

템플릿은 자신이 어떤 빈칸을 가졌는지 **`input_variables`**로 알고 있음. 또한 일부 변수를 미리 고정해 두는 **`partial`(부분 적용)** 기능이 있음.

```python
template = PromptTemplate.from_template("{product}를 {tone} 톤으로 소개해줘")
print(template.input_variables)   # → ['product', 'tone']

# 일부 변수만 미리 고정 (tone은 항상 '감성적인')
emotional = template.partial(tone="감성적인")
print(emotional.invoke({"product": "린넨 셔츠"}).text)
# → 린넨 셔츠를 감성적인 톤으로 소개해줘
```

### 비유

`partial`은 **도장에 미리 새겨 둔 회사 로고**와 같음. 매번 찍는 도장(템플릿)에 회사 로고(고정 변수)는 이미 박혀 있고, 가변 정보(상품명)만 그때그때 채우면 됨.

### 원리

`.invoke()`에 넘긴 딕셔너리의 키가 `input_variables`와 맞아야 함. 빠뜨리면 "변수가 비었다"는 오류가 남. `partial`은 그중 일부를 미리 채워, 남은 변수만 나중에 넣도록 템플릿을 새로 만들어 돌려줌. 자주 쓰는 고정값(현재 날짜, 브랜드명 등)을 partial로 박아 두면 호출이 간결해짐.

### 📌 참고
> `partial`에는 값 대신 **함수**도 넣을 수 있음. 예를 들어 "현재 시각"처럼 호출할 때마다 달라져야 하는 값은, 고정 문자열이 아니라 함수를 partial로 걸어 매번 새로 계산되게 함.

### 실무 포인트
- 변수 누락 오류는 초급자가 가장 자주 만나는 에러임. `input_variables`를 찍어 보면 무엇을 채워야 하는지 바로 확인됨.
- 브랜드명·기본 톤처럼 거의 안 바뀌는 값은 `partial`로 고정해 두면 호출부가 깔끔해짐.

### 3줄 요약
1. 템플릿은 `input_variables`로 자신의 빈칸 목록을 알고 있음.
2. `.invoke()` 딕셔너리 키는 이 목록과 맞아야 하며, 누락 시 오류가 남.
3. `partial`로 일부 변수(값 또는 함수)를 미리 고정할 수 있음.

---

## 2-5. Few-shot 프롬프팅 — 예시로 형식을 가르치기

### 개념

모델에게 **"이렇게 답해라"고 예시 몇 개를 보여주면** 출력 형식과 스타일을 훨씬 잘 따름. 이것이 **few-shot(소수 예시) 프롬프팅**임. 예시 없이 지시만 하는 것은 **zero-shot**임.

LangChain은 예시들을 깔끔하게 끼워 넣는 `FewShotPromptTemplate`을 제공함.

```python
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

# 1) 예시 하나를 어떻게 표현할지 정하는 틀
example_prompt = PromptTemplate.from_template("상품: {product}\n문구: {copy}")

# 2) 예시 데이터
examples = [
    {"product": "베이지 니트", "copy": "포근함을 입다."},
    {"product": "린넨 셔츠", "copy": "여름을 가볍게."},
]

# 3) 예시 + 새 질문을 합치는 템플릿
few_shot = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    suffix="상품: {product}\n문구:",   # 마지막에 붙는, 모델이 채울 부분
    input_variables=["product"],
)

print(few_shot.invoke({"product": "울 코트"}).text)
```

완성된 프롬프트는 예시 두 개를 보여준 뒤 "울 코트"의 문구를 모델이 같은 형식으로 채우게 유도함.

### 비유

few-shot은 **신입에게 "이렇게 쓰면 돼"라며 견본 몇 장을 보여주는 것**과 같음. 말로 백 번 설명하는 것보다, 잘 된 예시 두세 개를 보여주는 편이 형식을 훨씬 정확히 전달함.

### 원리

`FewShotPromptTemplate`은 `examples`의 각 딕셔너리를 `example_prompt` 틀에 하나씩 끼워 예시 블록들을 만든 뒤, 그 블록들을 이어 붙이고 마지막에 `suffix`(모델이 이어서 채울 부분)를 붙여 하나의 긴 프롬프트를 완성함. 모델은 앞의 예시 패턴을 보고 그 형식대로 뒤를 이어 씀.

### 📌 참고
> 채팅 모델용으로는 `FewShotChatMessagePromptTemplate`을 씀(메시지 형태로 예시를 넣음). 예시가 많아 일부만 골라 넣고 싶을 땐 `ExampleSelector`로 입력과 가장 비슷한 예시를 자동 선별함. 지금은 "예시를 보여주면 형식을 잘 따른다"는 핵심만 잡으면 됨.

### 실무 포인트
- 출력 형식이 까다롭거나(특정 JSON, 표 형식 등) 톤을 일정하게 유지해야 할 때 few-shot이 특히 강력함.
- 예시는 **질 좋은 2~5개**가 양보다 중요함. 나쁜 예시를 넣으면 모델이 그 나쁜 형식을 따라 함.

### 3줄 요약
1. few-shot은 예시 몇 개를 보여줘 출력 형식·스타일을 가르치는 기법임.
2. `FewShotPromptTemplate`은 예시 틀 + 예시 데이터 + suffix로 긴 프롬프트를 조립함.
3. 잘 고른 예시 2~5개가 형식 통제에 매우 효과적임.

---

## Part 2 마무리 — 다음으로

Part 2에서는 질문을 "찍어내는" 틀을 손에 쥐었음. 정리하면:

- 문자열 직접 조립의 한계 → 빈칸 양식지(템플릿)의 필요성 (2-1)
- 단일 문자열용 `PromptTemplate`과 `from_template`·`{변수}` 문법 (2-2)
- 메시지용 `ChatPromptTemplate`과 `("system", "...")` 튜플 문법 (2-3)
- `input_variables`로 빈칸을 확인하고 `partial`로 일부 고정 (2-4)
- 예시로 형식을 가르치는 few-shot 프롬프팅 (2-5)

여기까지 우리는 ① 모델 블록(Part 1)과 ② 프롬프트 템플릿 블록(Part 2)이라는 **두 개의 Runnable 블록**을 갖게 됐음. 둘 다 `.invoke()`를 가졌다는 공통점이 있음. **Part 3**에서는 드디어 이 두 블록을 파이프(`|`) 하나로 이어 붙임 — `프롬프트 | 모델`. 이것이 LangChain의 심장인 **LCEL(LangChain Expression Language)**이며, 초급의 정점임.

> 함께 제공되는 **Part 2 실습 노트북**에서는 `PromptTemplate`과 `ChatPromptTemplate`을 직접 만들어 변수를 채워 호출하고, `partial`로 브랜드명을 고정하며, few-shot으로 상품 홍보 문구의 형식을 모델에게 학습시키는 실습을 Gemini로 돌려봄.
