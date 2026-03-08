---
title: "프롬프트 엔지니어링: 이론부터 실전 서비스 구축까지 (3)"
date: 2026-03-08 13:00
excerpt: 자란(Jalan) 일본어 숙박 리뷰를 GPT로 요약하며, GPT-4의 출력을 One-shot 예시로 활용해 GPT-3.5의 품질을 끌어올린 과정을 정리했다.
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

# LLM을 활용한 숙박 리뷰 요약 - 모델 비교와 프롬프트 엔지니어링

## 들어가며

숙박 플랫폼에는 수많은 리뷰가 쌓인다. 이 리뷰들을 한눈에 파악할 수 있도록 요약해주면 사용자 경험이 크게 개선될 것이다. 이번 글에서는 OpenAI의 GPT 모델을 활용해 일본어 숙박 리뷰를 요약하고, **프롬프트 엔지니어링만으로 저렴한 모델의 품질을 끌어올린 과정**을 공유한다.

핵심 흐름은 다음과 같다:

> 단순 프롬프트 → GPT-3.5 (4점), GPT-4 (5점)
> → 상세 프롬프트 적용 → GPT-3.5 (2점으로 하락)
> → GPT-4 출력을 One-shot 예시로 활용 → GPT-3.5 (5점 달성)


## 1. 쟈란 리뷰 크롤링

요약할 리뷰 데이터를 확보하기 위해 일본 숙박 예약 플랫폼 쟈란(jalan.net)에서 리뷰를 크롤링했다. Selenium으로 페이지를 렌더링하고 BeautifulSoup으로 파싱하는 방식이다.

```python
from bs4 import BeautifulSoup
from selenium import webdriver

driver = webdriver.Chrome()
driver.get(url)

soup = BeautifulSoup(driver.page_source, 'html.parser')
containers = soup.select('.jlnpc-kuchikomiCassette')
```

각 리뷰 카세트(`.jlnpc-kuchikomiCassette`)에서 다음 정보를 추출한다:

- **리뷰 본문** (`.jlnpc-kuchikomiCassette__postBody`)
- **총점** (`.jlnpc-kuchikomiCassette__totalRate`)
- **투숙 날짜** ("2025年12月" 형식)
- **세부 평점** - 部屋(객실), 風呂(온천), 料理(식사), 接客・サービス(서비스), 清潔感(청결도)

세부 평점은 `dt/dd` 쌍으로 구성된 평점 리스트에서 추출하고, 이 방식이 실패할 경우 텍스트에서 정규식으로 폴백 추출한다.

```python
def extract_ratings(container):
    ratings = {}
    rate_list = container.select_one('.jlnpc-kuchikomiCassette__rateList')
    if rate_list:
        dts = rate_list.find_all('dt')
        for dt in dts:
            label = dt.get_text(strip=True)
            dd = dt.find_next_sibling('dd')
            # RATING_MAP으로 일본어 라벨 → 영문 필드명 매핑
            ...
    return ratings
```

페이지네이션은 "次へ"(다음) 버튼을 클릭하여 최대 5페이지까지 순회한다. 3개 숙박시설의 리뷰를 각각 JSON 파일로 저장했다.

## 2. 리뷰 데이터 전처리

먼저 JSON 형태의 리뷰 데이터를 전처리한다. 최근 6개월 이내의 리뷰만 필터링하고, 별점 5점(긍정)과 그 외(부정)로 분류했다. 각 리뷰는 `[REVIEW_START]`와 `[REVIEW_END]` 태그로 감싸서 LLM이 리뷰의 경계를 명확히 인식할 수 있도록 했다.

```python
for r in review_list:
    # 최근 6개월 필터링
    if review_date < date_boundary:
        continue

    if r['star'] == 5:
        reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
    else:
        reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
```

## 3. Baseline 프롬프트로 모델 비교

가장 단순한 프롬프트부터 시작했다.

```
아래 숙박시설 리뷰를 5줄 정도 분량으로 정리해주세요:
```

### GPT-3.5의 결과

GPT-3.5는 개별 리뷰를 번호를 매겨 나열하는 방식으로 요약했다. 각 리뷰의 내용을 축약하긴 했지만, **통합적인 요약이라기보다는 리뷰 발췌에 가까운 결과**였다.

### GPT-4의 결과

반면 GPT-4는 테마별로 정보를 통합했다. 식사, 스파, 스태프 대응, 객실, 혼잡도 등 카테고리로 나누어 전체 리뷰의 경향을 하나의 요약으로 정리했다.

### G-Eval로 품질 평가

요약의 품질을 정량적으로 비교하기 위해 [G-Eval](https://arxiv.org/abs/2303.16634) 논문의 평가 방식을 적용했다. Coherence(일관성) 지표를 1~5점 척도로 평가한다.

```python
evaluation_prompt = f"""
Evaluation Criteria:
Coherence (1-5): The collective quality of all sentences.
A high-quality summary should be well-structured and organized,
and not just a heap of related sentences.
...
Source Text: {document}
Summary: {summary}
"""
```

| 모델 | Coherence 점수 | 평가 요약 |
|------|:-:|------|
| GPT-3.5 | 4 | 번호별로 정리되어 있지만, 통합된 하나의 요약이 아닌 개별 미니 요약의 나열 |
| GPT-4 | 5 | 테마별로 잘 구조화되어 있고, 일관된 흐름을 가짐 |

GPT-4가 확실히 더 나은 결과를 보여주지만, 비용 차이가 크다. GPT-3.5의 품질을 끌어올릴 방법이 없을까?

## 4. 상세 프롬프트 - 오히려 역효과

프롬프트에 구체적인 조건을 추가해봤다.

```
당신은 요약 전문가입니다.

요약 결과는 다음 조건을 충족해야 합니다:
1. 모든 문장은 반드시 정중어(경어체)로 끝낼 것
2. 숙박시설을 소개하는 톤앤매너로 작성할 것
3. 요약 결과는 최소 2문장, 최대 5문장으로 작성할 것
```

좋은 예와 나쁜 예까지 제시하며 톤앤매너를 명확히 지정했다. 그런데 결과는 예상 밖이었다.

**GPT-3.5의 Coherence 점수: 2점**

GPT-3.5는 지시사항을 제대로 따르지 못했다. 요약이 아닌 원문 리뷰를 거의 그대로 복사했고, 문장 끝에 "멋졌었습니다입니다", "생각했습니다입니다" 같은 어색한 표현이 나타났다. 조건이 복잡해지자 오히려 기본적인 요약 능력까지 저하된 것이다.

이는 중요한 교훈을 준다: **모델의 능력을 초과하는 복잡한 지시는 역효과를 낼 수 있다.**

## 5. One-shot 프롬프트 (1차 시도) - 동일 데이터의 함정

여기서 핵심 아이디어가 등장한다. GPT-4로 먼저 좋은 요약 예시를 생성하고, 이를 GPT-3.5의 프롬프트에 one-shot 예시로 넣는 것이다.

```python
# 1단계: GPT-4로 예시 요약 생성
summary_1shot = summarize(reviews, PROMPT_ONE_SHOT,
                          model='gpt-4-turbo-2024-04-09')

# 2단계: 생성된 예시를 GPT-3.5 프롬프트에 포함
prompt_1shot = f"""당신은 요약 전문가입니다.
...
아래는 리뷰와 요약의 예시입니다.
예시 리뷰:
{reviews_1shot}
예시 요약 결과:
{summary_1shot}

아래 숙박시설 리뷰를 요약해주세요:"""

# 3단계: GPT-3.5로 실행
result = summarize(reviews, prompt_1shot,
                   temperature=1.0, model='gpt-3.5-turbo-0125')
```

### 결과: Coherence 5점, 하지만 문제가 있다

G-Eval Coherence 점수는 5점을 달성했지만, 실제 출력을 보면 문제가 드러난다:

> 화실 호반 쪽 객실에서의 경치가 훌륭하고, 청결감 넘치는 공간에서 석식과 조식 뷔페가 매우 만족스러웠습니다. 특히 아이들은 옥상 스파에서 매우 만족했으며, 소중한 가족 시간을 보낼 수 있었습니다. 스태프 대응도 따뜻하고, 좋은 여행이 되었습니다.

이 결과는 **리뷰 요약이 아니라 실제 리뷰를 쓰는 것처럼** 되어 있다. "~매우 만족스러웠습니다", "~좋은 여행이 되었습니다"는 투숙객 본인의 감상이지, "~라는 평가입니다", "~라는 의견이 많습니다" 같은 요약 톤이 아니다.

원인은 **one-shot 예시의 리뷰와 실제 요약 대상 리뷰가 동일한 데이터**였기 때문이다. `reviews_1shot`과 `reviews` 모두 같은 `reviews.json`에서 가져왔기 때문에, 모델은 예시의 톤을 학습하는 것이 아니라 이미 본 리뷰 내용을 그대로 반복해버린 것이다.

## 6. One-shot 프롬프트 (2차 시도) - 별도 예시로 해결

문제를 해결하기 위해 두 가지를 변경했다:

1. **one-shot 예시를 별도의 리뷰/요약 쌍으로 하드코딩** - 요약 대상과 완전히 다른 데이터를 사용
2. **프롬프트에 요약 톤과 구조를 명시** - "~라는 평가입니다" 패턴과 요약 순서를 지정

```python
# 요약 대상과 다른 별도의 예시 데이터
example_reviews = """[REVIEW_START]방이 넓고 쾌적했습니다. 방음도 잘 되어 있어서...[REVIEW_END]
[REVIEW_START]역에서 가깝고 접근성이 좋았습니다. 프론트 직원도 친절하고...[REVIEW_END]
..."""

# 원하는 톤의 요약 예시를 직접 작성
example_summary = """방이 넓고 청결하며, 방음도 잘 되어 있다는 평가가 많습니다.
역에서의 접근성이 좋고, 프론트 대응도 친절하다는 의견이 많이 보입니다...."""

prompt_1shot = f"""당신은 요약 전문가입니다.
...
4. 개별 리뷰를 그대로 쓰는 것이 아니라, 여러 리뷰의 경향을 종합하여
   "~라는 평가입니다", "~라는 의견이 많습니다"와 같이 작성해주세요.
5. 요약은 "시설·객실 → 식사 → 서비스·스태프 → 종합 평가" 순서로
   구성해주세요.

아래는 리뷰와 요약의 예시입니다.
예시 리뷰:
{example_reviews}
예시 요약 결과:
{example_summary}

아래 숙박시설 리뷰를 요약해주세요:"""
```

### 결과: temperature=1.0에서도 Coherence 5점

`temperature=1.0`의 높은 랜덤성에서도 안정적으로 5점을 달성했다. 별도 예시와 명확한 구조 지시가 효과를 발휘한 것이다.

핵심 차이는 출력의 톤에 있다. 1차 시도에서 "매우 만족스러웠습니다"(직접 체험한 감상)로 나왔던 것이, 2차 시도에서는 "~라는 평가가 많습니다"(리뷰 경향 요약)로 바뀌었다.

최종 요약 결과:

> 종합적으로 훌륭한 숙박시설이라는 평가입니다. 객실은 청결하고 경치도 좋아, 가족 단위나 친구끼리 이용하기에 최적입니다. 석식, 조식 모두 메뉴가 풍부하고, 특히 천공 스파에 대한 만족도가 매우 높습니다. 또한 프론트와 스태프의 대응도 좋아, 쾌적한 여행이 되었다는 의견이 많습니다. 재방문하고 싶다는 목소리도 많이 보입니다.

## 정리

| 방식 | 모델 | Coherence | 비고 |
|------|------|:-:|------|
| Baseline 프롬프트 | GPT-3.5 | 4 | 개별 리뷰 나열 |
| Baseline 프롬프트 | GPT-4 | 5 | 테마별 통합 요약 |
| 상세 조건 프롬프트 | GPT-3.5 | 2 | 지시 과부하로 품질 저하 |
| One-shot (동일 데이터 예시) | GPT-3.5 | 5 | 점수는 높지만 리뷰 톤으로 출력 |
| One-shot (별도 데이터 예시) | GPT-3.5 | 5 | 요약 톤으로 올바르게 출력 |

이번 실험에서 얻은 시사점은 다음과 같다:

1. **복잡한 지시보다 좋은 예시가 낫다** - 모델 능력을 초과하는 조건을 나열하는 것보다, 원하는 결과의 예시를 하나 보여주는 것이 훨씬 효과적이다.
2. **one-shot 예시와 실제 입력은 반드시 다른 데이터를 사용해야 한다** - 동일 데이터를 사용하면 모델이 패턴을 학습하지 못하고 내용을 그대로 반복한다. 예시는 "어떤 톤으로 요약할지"를 가르치는 것이지, 내용을 복사하라는 신호가 아니다.
3. **정량 평가만으로는 부족하다** - Coherence 점수가 5점이어도 실제 출력의 톤이 의도와 다를 수 있다. G-Eval은 구조적 일관성을 평가하지만, 톤앤매너의 적절성은 별도로 확인해야 한다.
4. **평가 자동화가 중요하다** - G-Eval 같은 LLM 기반 평가를 도입하면 프롬프트 변경의 효과를 정량적으로 비교할 수 있어, 감에 의존하지 않는 개선이 가능하다.
