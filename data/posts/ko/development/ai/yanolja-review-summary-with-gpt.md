---
title: "프롬프트 엔지니어링: 이론부터 실전 서비스 구축까지 (2)"
date: 2026-03-01 13:54
excerpt: 야놀자 숙소 리뷰를 크롤링하고 GPT-3.5-Turbo로 자동 요약하는 파이프라인을 구축하며, Few-shot Prompting과 Pairwise Evaluation으로 품질을 개선한 과정을 정리했다.
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

# GPT를 활용한 야놀자 숙소 리뷰 자동 요약 서비스 구축기

숙소를 예약할 때 수백 개의 리뷰를 일일이 읽어보는 건 꽤 번거로운 일이다. "이 숙소 리뷰 요약 좀 해줘"라는 단순한 아이디어에서 출발해, 야놀자 숙소 리뷰를 크롤링하고 GPT로 자동 요약하는 파이프라인을 구축해본 과정을 정리한다.

---

## 전체 파이프라인 개요

```
[크롤링] → [전처리] → [프롬프트 엔지니어링] → [요약 생성] → [평가] → [데모 배포]
```

1. **크롤링**: Selenium + BeautifulSoup으로 야놀자 리뷰 수집
2. **전처리**: 날짜/길이 필터링, 긍정·부정 리뷰 분리
3. **요약**: GPT-3.5-Turbo + Few-shot Prompting
4. **평가**: MT-Bench 스타일 Pairwise Evaluation (GPT-4o)
5. **데모**: Gradio 웹 인터페이스

---

## 1. 데이터 수집 — 야놀자 리뷰 크롤링

야놀자 리뷰 페이지는 스크롤 시 동적으로 리뷰를 로딩하기 때문에, 단순 HTTP 요청만으로는 데이터를 가져올 수 없다. Selenium으로 브라우저를 제어해 스크롤을 반복한 뒤, BeautifulSoup으로 HTML을 파싱했다.

```python
def crawl_yanolja_reviews(name, url):
    review_list = []
    driver = webdriver.Chrome()
    driver.get(url)

    time.sleep(3)

    # 스크롤을 반복해 동적 로딩된 리뷰를 모두 불러옴
    scroll_count = 20
    for i in range(scroll_count):
        driver.execute_script('window.scrollTo(0, document.body.scrollHeight);')
        time.sleep(2)

    html = driver.page_source
    soup = BeautifulSoup(html, 'html.parser')

    review_containers = soup.select(
        '#__next > section > div > div.css-1js0bc8 > div > div > div'
    )

    for i in range(len(review_containers)):
        review_text = review_containers[i].find('p', class_='content-text').text
        review_stars = review_containers[i].select('path[fill="currentColor"]')
        star_cnt = sum(1 for star in review_stars if not star.has_attr('fill-rule'))
        date = review_date[i].text

        review_list.append({
            'review': review_text,
            'stars': star_cnt,
            'date': date
        })

    with open(f'./res/{name}.json', 'w') as f:
        json.dump(review_list, f, indent=4, ensure_ascii=False)
```

수집한 데이터 형태는 다음과 같다.

```json
[
    {"review": "깨끗하고 너무 좋았어요", "stars": 5, "date": "22시간 전"},
    {"review": "위치도 아주좋고 숙소도 깨끗하고 좋아요~", "stars": 5, "date": "2일 전"},
    ...
]
```

별점 카운팅 로직이 조금 독특한데, 야놀자에서 별을 SVG `<path>`로 렌더링하기 때문에 `fill="currentColor"` 속성이 있으면서 `fill-rule` 속성이 없는 path 요소의 개수를 세는 방식으로 별점을 추출했다.

인사동, 판교, 용산 3개 지점의 리뷰를 수집하여 실험에 활용했다.

---

## 2. 데이터 전처리

수집한 리뷰를 그대로 모델에 넣으면 노이즈가 많아 요약 품질이 떨어진다. 다음 기준으로 전처리를 수행했다.

```python
def preprocess_reviews(path='./res/reviews.json'):
    with open(path, 'r', encoding='utf-8') as f:
        review_list = json.load(f)

    reviews_good, reviews_bad = [], []

    current_date = datetime.datetime.now()
    date_boundary = current_date - datetime.timedelta(days=6*30)

    for r in review_list:
        review_date = parser.parse(r['date'])

        # 1) 최근 6개월 이내 리뷰만 사용
        if review_date < date_boundary:
            continue
        # 2) 30자 미만의 짧은 리뷰 제거
        if len(r['review']) < 30:
            continue

        # 3) 별점 기준으로 긍정/부정 분리
        if r['stars'] == 5:
            reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
        else:
            reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')

    # 4) 최대 50개로 제한
    reviews_good = reviews_good[:50]
    reviews_bad = reviews_bad[:50]

    return '\n'.join(reviews_good), '\n'.join(reviews_bad)
```

### 전처리 기준

| 기준 | 이유 |
|------|------|
| 최근 6개월 이내 | 오래된 리뷰는 현재 숙소 상태를 반영하지 못함 |
| 30자 이상 | "좋아요", "깨끗해요" 같은 짧은 리뷰는 요약에 도움이 안 됨 |
| 별점 5점 vs 4점 이하 | 긍정/부정 요약을 별도로 생성하기 위해 분리 |
| 최대 50개 | 토큰 한도 내에서 처리하기 위한 제한 |

각 리뷰는 `[REVIEW_START]`와 `[REVIEW_END]` 구분자로 감싸서 모델이 개별 리뷰의 경계를 명확히 인식할 수 있도록 했다.

실제 인사동 데이터 기준으로, 전체 220개 리뷰 중 5점 리뷰 177개, 4점 이하 리뷰 43개로 분포가 확인되었다.

---

## 3. 프롬프트 엔지니어링 — 반복적 개선 과정

프롬프트 엔지니어링은 이 프로젝트의 핵심이었다. Baseline부터 시작해 4단계에 걸쳐 프롬프트를 개선했고, 각 단계마다 평가를 통해 효과를 검증했다.

### 3-1. Baseline — 단순 지시

```python
PROMPT_BASELINE = "아래 숙소 리뷰에 대해 5문장 내로 요약해줘:"
```

가장 단순한 프롬프트로 시작했다. GPT-3.5-Turbo에게 별다른 조건 없이 요약을 요청한 것이다.

**결과**: 10전 10승 (vs. 야놀자 실제 요약)

의외로 단순한 프롬프트도 기존 야놀자 요약보다 나은 결과를 보여줬다. 하지만 출력 형식이 일관되지 않고, 반말과 존댓말이 섞이는 문제가 있었다.

### 3-2. 고도화 1 — 출력 조건 명시

```python
prompt = """당신은 요약 전문가입니다. 사용자 숙소 리뷰들이 주어졌을 때 요약하는 것이 당신의 목표입니다.

요약 결과는 다음 조건들을 충족해야 합니다:
1. 모든 문장은 항상 존댓말로 끝나야 합니다.
2. 숙소에 대해 소개하는 톤앤매너로 작성해주세요.
  2-1. 좋은 예시
    a) 전반적으로 좋은 숙소였고 방음도 괜찮았다는 평입니다.
    b) 재방문 예정이라는 평들이 존재합니다.
  2-2. 나쁜 예시
    a) 좋은 숙소였고 방음도 괜찮았습니다.
    b) 재방문 예정입니다.
3. 요약 결과는 최소 2문장, 최대 5문장 사이로 작성해주세요.

아래 숙소 리뷰들에 대해 요약해주세요:"""
```

역할 부여, 존댓말, 톤앤매너, 문장 수 제한 등 구체적인 조건을 추가했다. 특히 "소개하는 톤앤매너"를 좋은 예시/나쁜 예시로 구분해 설명한 것이 포인트다. 리뷰를 그대로 옮기는 것이 아니라, 제3자가 숙소를 소개하는 느낌의 문체를 유도했다.

**결과**: 10전 10승 (vs. 야놀자 실제 요약)

### 3-3. 고도화 2 — 입력 데이터 품질 향상

프롬프트가 아닌 입력 데이터 쪽을 개선했다. 앞서 설명한 전처리 로직(30자 미만 제거, 6개월 이내, 50개 제한)을 이 단계에서 적용했다.

**결과**: 10전 7승 3패 (vs. 야놀자 실제 요약)

흥미롭게도 성능이 오히려 떨어졌다. 짧지만 핵심 키워드를 담고 있던 리뷰들("위치 최고", "깨끗해요")이 제거되면서 요약의 다양성이 감소한 것으로 분석했다. 하지만 장기적으로 노이즈 제거는 필요한 과정이므로, 전처리는 유지하고 프롬프트 쪽에서 보완하는 방향으로 진행했다.

### 3-4. 고도화 3 — Few-Shot Prompting

성능 하락을 만회하기 위해 Few-shot Prompting을 도입했다. GPT-4-Turbo로 판교 지점 리뷰의 "모범 요약"을 생성하고, 이를 예시로 함께 제공하는 방식이다.

```python
# GPT-4-Turbo로 판교 리뷰의 모범 요약 생성
reviews_1shot, _ = preprocess_reviews('./res/ninetree_pangyo.json')
summary_1shot = summarize(
    reviews_1shot, prompt, temperature=0.0, model='gpt-4-turbo-2024-04-09'
).choices[0].message.content

# 1-Shot 프롬프트 구성
prompt_1shot = f"""당신은 요약 전문가입니다. 사용자 숙소 리뷰들이 주어졌을 때
요약하는 것이 당신의 목표입니다. 다음은 리뷰들과 요약 예시입니다.

예시 리뷰들:
{reviews_1shot}
예시 요약 결과:
{summary_1shot}

아래 숙소 리뷰들에 대해 요약해주세요:"""
```

핵심 아이디어는 **GPT-4의 출력을 GPT-3.5의 가이드로 사용**하는 것이다. 비싼 GPT-4는 예시 생성에만 한 번 사용하고, 실제 서비스에서는 비용 효율적인 GPT-3.5를 사용한다.

**1-Shot 결과**: 10전 9승 1패 (vs. 야놀자 실제 요약)

용산 지점 리뷰로 2-Shot까지 확장해봤다.

**2-Shot 결과**: 10전 9승 1패 (vs. 야놀자 실제 요약)

1-Shot과 2-Shot의 성능 차이가 없었기 때문에 최종적으로 1-Shot을 채택했다. 토큰 비용 대비 성능 향상이 없다면 더 적은 예시를 쓰는 것이 합리적이다.

### 프롬프트 개선 과정 요약

| 버전 | 전략 | 승/패/무 | 비고 |
|------|------|----------|------|
| Baseline | 단순 지시 | 10/0/0 | 형식 일관성 부족 |
| v1 | 조건 명시 | 10/0/0 | 톤앤매너 개선 |
| v2 | 전처리 강화 | 7/3/0 | 데이터 품질↑, 다양성↓ |
| v3 (1-Shot) | Few-Shot | 9/1/0 | GPT-4 예시 활용 |
| v4 (2-Shot) | Few-Shot | 9/1/0 | 1-Shot 대비 차이 없음 |

---

## 4. 평가 방법 — MT-Bench 스타일 Pairwise Evaluation

LLM이 생성한 요약의 품질을 어떻게 평가할 것인가? BLEU, ROUGE 같은 전통적 메트릭은 텍스트의 의미적 품질을 제대로 반영하지 못한다. 이 프로젝트에서는 MT-Bench 논문에서 제안한 **Pairwise Comparison** 방식을 채택했다.

### 왜 점수 매기기가 아닌 비교 방식인가?

- **점수 방식의 문제**: "이 요약은 몇 점인가?" → 기준이 모호함 (3점짜리 요약이 뭔데?)
- **비교 방식의 장점**: "A와 B 중 어느 쪽이 더 나은가?" → 상대적 판단이 더 명확하고 일관됨

### 평가 구현

GPT-4o를 심판(Judge)으로 사용하여 두 요약을 비교한다.

```python
def pairwise_eval(reviews, answer_a, answer_b):
    eval_prompt = f"""[System]
Please act as an impartial judge and evaluate the quality of the Korean
summaries provided by two AI assistants to the set of user reviews on
accommodations displayed below. You should choose the assistant that
follows the user's instructions and answers the user's question better.
Your evaluation should consider factors such as the helpfulness, relevance,
accuracy, depth, creativity, and level of detail of their responses.
...
After providing your explanation, output your final verdict by strictly
following this format: "[[A]]" if assistant A is better, "[[B]]" if
assistant B is better, and "[[C]]" for a tie.

[User Reviews]
{reviews}
[The Start of Assistant A's Answer]
{answer_a}
[The End of Assistant A's Answer]
[The Start of Assistant B's Answer]
{answer_b}
[The End of Assistant B's Answer]"""

    completion = client.chat.completions.create(
        model='gpt-4o-2024-05-13',
        messages=[{'role': 'user', 'content': eval_prompt}],
        temperature=0.0
    )
    return completion
```

### 평가 기준

GPT-4o가 판단하는 기준은 다음 6가지다.

- **Helpfulness**: 사용자에게 실질적으로 도움이 되는가
- **Relevance**: 리뷰 내용과 관련 있는가
- **Accuracy**: 사실에 기반한 요약인가
- **Depth**: 충분한 깊이를 가지고 있는가
- **Creativity**: 단순 나열이 아닌 종합적 요약인가
- **Level of Detail**: 적절한 수준의 디테일을 포함하는가

### 대규모 평가 — temperature 활용

동일한 프롬프트에서도 다양한 출력을 얻기 위해 `temperature=1.0`으로 10개의 요약을 생성하고, 각각을 야놀자 실제 요약과 비교하여 승률을 측정했다.

```python
def pairwise_eval_batch(reviews, answers_a, answers_b):
    a_cnt, b_cnt, draw_cnt = 0, 0, 0
    for i in range(len(answers_a)):
        completion = pairwise_eval(reviews, answers_a[i], answers_b[i])
        verdict_text = completion.choices[0].message.content

        if '[[A]]' in verdict_text:
            a_cnt += 1
        elif '[[B]]' in verdict_text:
            b_cnt += 1
        elif '[[C]]' in verdict_text:
            draw_cnt += 1

    return a_cnt, b_cnt, draw_cnt
```

---

## 5. 요약 생성

최종 요약 생성은 GPT-3.5-Turbo를 `temperature=0.0`으로 호출하여 일관된 결과를 보장한다.

```python
def summarize(reviews):
    prompt = PROMPT + '\n\n' + reviews

    client = OpenAI(api_key=OPENAI_API_KEY)
    completion = client.chat.completions.create(
        model='gpt-3.5-turbo-0125',
        messages=[{'role': 'user', 'content': prompt}],
        temperature=0.0
    )
    return completion
```

긍정 리뷰와 부정 리뷰를 각각 별도로 요약하여, 사용자가 숙소의 장점과 단점을 한눈에 파악할 수 있도록 했다.

---

## 6. Gradio 데모

Gradio를 사용해 간단한 웹 인터페이스를 구성했다.

```python
def run_demo():
    demo = gr.Interface(
        fn=fn,
        inputs=[gr.Radio(['인사동', '판교', '용산'], label='숙소')],
        outputs=[
            gr.Textbox(label='높은 평점 요약'),
            gr.Textbox(label='낮은 평점 요약')
        ]
    )
    demo.launch(share=True)
```

사용자가 숙소를 선택하면 크롤링된 리뷰를 전처리하고, GPT로 긍정/부정 요약을 각각 생성하여 보여준다. `share=True`로 외부 공유 링크도 생성할 수 있다.

---

## 배운 점

**데이터 품질과 프롬프트는 트레이드오프 관계가 될 수 있다.**
전처리를 강화하면 노이즈는 줄어들지만 정보량도 감소한다. 이때 Few-shot Prompting이 효과적인 보완 수단이 됐다.

**GPT-4의 출력을 GPT-3.5의 가이드로 쓰는 전략이 비용 효율적이다.**
GPT-4로 모범 답안을 한 번 생성해두고, 실제 서비스에서는 GPT-3.5로 운영하면 품질과 비용을 모두 잡을 수 있다.

**LLM 평가에는 Pairwise Comparison이 직관적이다.**
점수 기반 평가보다 "A vs B 중 어느 쪽이 나은가"라는 비교 방식이 더 일관되고 해석하기 쉬웠다.

**Few-shot은 1개면 충분할 수 있다.**
1-Shot과 2-Shot의 성능 차이가 없었다. 예시 수를 무작정 늘리는 것보다, 좋은 예시 하나를 잘 고르는 것이 중요하다.

---

## 사용 기술 스택

| 구분 | 기술 |
|------|------|
| 크롤링 | Selenium, BeautifulSoup |
| 요약 모델 | GPT-3.5-Turbo (서비스), GPT-4-Turbo (예시 생성) |
| 평가 모델 | GPT-4o |
| 프롬프트 | Few-shot Prompting (1-Shot) |
| 데모 | Gradio |
| 언어 | Python |
