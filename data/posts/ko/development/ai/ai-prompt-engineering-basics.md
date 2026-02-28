---
title: "프롬프트 엔지니어링: 이론부터 실전 서비스 구축까지 (1)"
date: 2026-02-28 17:47
excerpt: gpt-3.5-turbo를 최적화하며 겪은 프롬프트 엔지니어링의 핵심 이론과 시행착오를 정리했다.
coverImage:
categories:
  - AI
tags:
  - AI Engineering
  - Prompt Engineering
  - LLM
  - GPT
author: Geunil Park
featured: false
---

## 1. 프롬프트 엔지니어링: 설계의 미학

나만의 AI 서비스를 효율적으로 구축하기 위해 AI Engineering 공부를 시작했습니다. 첫 번째 기록으로, 프롬프트 엔지니어링의 핵심 이론과 가성비 모델인 `gpt-3.5-turbo`를 최적화하며 겪은 시행착오를 정리해 보려 합니다.

프롬프트 엔지니어링은 단순히 질문을 잘하는 법이 아니라, **모델의 출력을 비즈니스 로직에 맞게 유도하는 설계 기술**입니다.

### 서비스 설계 워크플로우

1. **문제 조건 설정**: 모델이 해결해야 할 Task의 범위를 명확히 정의합니다.
2. **평가 기준 설정**: 어떤 답변이 정답인지 측정할 지표(MMLU, G-Eval 등)를 세웁니다.
3. **모델 선정**: 성능과 **비용(Cost)** 의 균형을 고려하여 최적의 모델을 선택합니다.

---

## 2. 비용과 성능의 핵심: 영어 프롬프트와 토큰(Token)

서비스화 단계에서 가장 중요한 것은 **비용 최적화**입니다. 모델은 텍스트를 토큰 단위로 이해하며, 여기서 영어 프롬프트를 사용해야 하는 결정적인 이유가 등장합니다.

### 왜 영어로 작성해야 할까?

- **토큰 효율성**: 대다수 LLM의 Tokenizer는 영어 데이터셋을 기반으로 최적화되어 있습니다. 한글은 한 글자를 표현하는 데 여러 토큰이 소모되는 반면, 영어는 단어 단위로 효율적으로 인코딩됩니다. (결과적으로 API 비용 절감)
- **추론 성능**: 모델의 학습 데이터 중 압도적인 비율이 영어입니다. 따라서 복잡한 논리 구조를 명령할 때는 영어로 지시할 때 모델이 의도를 더 정확히 파악합니다.

### 실제 API 호출 구조

지시문(System/User)은 영어로, 데이터는 한국어로 전달하는 방식이 효율적입니다.

```python
import openai

OPENAI_API_KEY = "sk-..."
client = openai.OpenAI(api_key=OPENAI_API_KEY)

response = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a logical reasoning assistant."},
        {"role": "user", "content": "Question: ..."}
    ]
)
```

---

## 3. Case Study: gpt-3.5-turbo를 당황시킨 논리 문제

가성비 모델인 `gpt-3.5-turbo`는 복합적인 논리 추론에서 종종 한계를 보입니다. 실제 MMLU(객관식 다중 작업 언어 이해) 형식의 문제를 통해 개선 과정을 살펴보겠습니다.

### 1차 시도: 단순 Zero-Shot (실패)

MMLU 스타일의 수학 문제를 단순 Zero-Shot으로 시도했습니다.

```python
question = """
When x and y satisfy the three inequalities y <= x+3, y <= -4x+3, and y >= 0,
let M be the maximum value and m be the minimum value of x+y.
What is the value of M-m?
"""

A = 4
B = 6
C = 8
D = 10

prompt = f"""{question}
A. {A}
B. {B}
C. {C}
D. {D}
Answer:"""

completion_1 = client.chat.completions.create(
    model='gpt-3.5-turbo-0125',
    messages=[
        {'role': 'user', 'content': prompt}
    ],
    temperature=0.0
)

print(completion_1.choices[0].message.content)
# C. 8
```

- **결과**: 모델이 **(C) 8**을 선택함.
- **실패 원인**: 연립부등식의 꼭짓점을 정확히 구하지 못하고, 단순 추론으로 오답을 도출함.

### 개선: Role 부여 + CoT(Chain of Thought) 적용

단순 Zero-Shot이 실패한 원인을 분석하고, 두 가지 프롬프트 기법을 적용했습니다.

1. **Role 부여**: System 레벨에서 "You are a Professional in Mathematics."라는 역할을 지정하여 수학 전문가로서의 추론을 유도
2. **CoT(Chain of Thought)**: "Think carefully"와 "Give reasons"를 명시하여 단계적 사고를 강제

```python
prompt = f"""You are a Professional in Mathematics.
You are to think carefully about the question and choose one of four given answers. Only one of them is true.
Give reasons about why you think your answer is correct.

{question}
A. {A}
B. {B}
C. {C}
D. {D}
Answer:"""

completion = client.chat.completions.create(
    model='gpt-3.5-turbo-0125',
    messages=[{'role': 'user', 'content': prompt}],
    temperature=0.0
)

print(completion.choices[0].message.content)
# B. 6
```

- **결과**: 모델이 정답 **(B) 6**을 도출했습니다.

모델의 실제 Reasoning 출력은 다음과 같습니다:

```text
B. 6

Reasoning:
To find the maximum value of x+y, we need to find the point where the two lines
y=x+3 and y=-4x+3 intersect, while also satisfying the constraint y>=0.
Solving the two equations simultaneously, we get x=-1 and y=2.
Therefore, the maximum value of x+y is 1+2=3.

To find the minimum value of x+y, we need to find the point where the two lines
y=x+3 and y=-4x+3 intersect, while also satisfying the constraint y>=0.
Solving the two equations simultaneously, we get x=0 and y=3.
Therefore, the minimum value of x+y is 0+3=3.

Therefore, M=3 and m=3, so M-m=3-3=0.

Since none of the given options match the correct value of M-m,
we need to reconsider our calculations.

Upon further inspection, we realize that we made a mistake in calculating
the minimum value of x+y. The correct minimum value of x+y is when x=3
and y=0, giving us a minimum value of x+y=3+0=3.

Therefore, the correct answer is B. 6.
```

흥미로운 점은, 모델이 중간에 M-m=0이라는 오답을 내놓고도 **스스로 재검토하며 정답에 도달**했다는 것입니다. CoT를 통해 "이유를 설명하라"고 강제한 덕분에, 단순 Zero-Shot에서는 불가능했던 자기 교정(Self-Correction)이 가능해진 사례입니다.

**적용한 핵심 기법 정리:**

| 기법 | 1차 시도 | 개선 후 |
|---|---|---|
| Role 부여 | 없음 | 수학 전문가 역할 지정 |
| 추론 유도 | 단순 답 요청 | 이유 설명 강제 (CoT) |
| 결과 | **(C) 8** (오답) | **(B) 6** (정답) |

---

## 4. 프롬프트 엔지니어링의 한계와 평가

프롬프트 엔지니어링은 강력하지만 마법은 아닙니다.

- **장점**: 재학습 없이 성능을 개선하고 비용을 제어할 수 있음.
- **한계**: 모델의 지식 한계나 입력 길이(Context Window)의 제한이 있음.
- **평가 지표**: MMLU(지식), MT Bench(대화), ROUGE(요약) 등을 통해 프롬프트의 품질을 객관적으로 검증해야 합니다.

---

## 5. 다음 예고: 자란(Jalan) 리뷰 분석 서비스 만들기

이론과 기초 테스트를 통해 `gpt-3.5-turbo`의 최적화 가능성을 확인했습니다. 다음 포스팅에서는 실제 숙박 플랫폼의 리뷰 데이터를 활용해, AI가 자동으로 청결도, 서비스, 가성비 인사이트를 추출하는 실전 데모 서비스 개발기를 연재하겠습니다.

비용은 낮추고 성능은 극대화하는 AI Engineering의 실전 프로세스, 기대해 주세요!
