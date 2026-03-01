---
title: "Prompt Engineering: From Theory to Building a Real Service (2)"
date: 2026-03-01 13:54
excerpt: I built a pipeline that crawls Yanolja accommodation reviews and automatically summarizes them using GPT-3.5-Turbo, improving quality through Few-shot Prompting and Pairwise Evaluation.
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

# Building an Automated Review Summary Service for Yanolja Accommodations Using GPT

Reading through hundreds of reviews when booking a hotel is quite tedious. Starting from the simple idea of "just summarize these accommodation reviews for me," I built a pipeline that crawls Yanolja accommodation reviews and automatically summarizes them with GPT. Here's the full process.

---

## Pipeline Overview

```
[Crawling] → [Preprocessing] → [Prompt Engineering] → [Summary Generation] → [Evaluation] → [Demo Deployment]
```

1. **Crawling**: Collecting Yanolja reviews with Selenium + BeautifulSoup
2. **Preprocessing**: Date/length filtering, separating positive and negative reviews
3. **Summarization**: GPT-3.5-Turbo + Few-shot Prompting
4. **Evaluation**: MT-Bench style Pairwise Evaluation (GPT-4o)
5. **Demo**: Gradio web interface

---

## 1. Data Collection — Crawling Yanolja Reviews

Yanolja's review pages dynamically load reviews on scroll, so simple HTTP requests alone can't retrieve the data. I used Selenium to control the browser and repeat scrolling, then parsed the HTML with BeautifulSoup.

```python
def crawl_yanolja_reviews(name, url):
    review_list = []
    driver = webdriver.Chrome()
    driver.get(url)

    time.sleep(3)

    # Repeat scrolling to load all dynamically loaded reviews
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

The collected data looks like this:

```json
[
    {"review": "Clean and really great", "stars": 5, "date": "22 hours ago"},
    {"review": "Great location and clean accommodation~", "stars": 5, "date": "2 days ago"},
    ...
]
```

The star rating counting logic is somewhat unique. Since Yanolja renders stars as SVG `<path>` elements, I extracted the star count by counting path elements that have the `fill="currentColor"` attribute but not the `fill-rule` attribute.

Reviews from three locations — Insadong, Pangyo, and Yongsan — were collected for the experiments.

---

## 2. Data Preprocessing

Feeding raw reviews directly into the model produces noisy results and poor summary quality. I applied the following preprocessing criteria:

```python
def preprocess_reviews(path='./res/reviews.json'):
    with open(path, 'r', encoding='utf-8') as f:
        review_list = json.load(f)

    reviews_good, reviews_bad = [], []

    current_date = datetime.datetime.now()
    date_boundary = current_date - datetime.timedelta(days=6*30)

    for r in review_list:
        review_date = parser.parse(r['date'])

        # 1) Only use reviews from the last 6 months
        if review_date < date_boundary:
            continue
        # 2) Remove short reviews under 30 characters
        if len(r['review']) < 30:
            continue

        # 3) Separate positive/negative based on star rating
        if r['stars'] == 5:
            reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
        else:
            reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')

    # 4) Limit to maximum 50 reviews
    reviews_good = reviews_good[:50]
    reviews_bad = reviews_bad[:50]

    return '\n'.join(reviews_good), '\n'.join(reviews_bad)
```

### Preprocessing Criteria

| Criterion | Reason |
|-----------|--------|
| Within last 6 months | Old reviews don't reflect the current state of the accommodation |
| 30+ characters | Short reviews like "nice" or "clean" don't help with summarization |
| 5 stars vs 4 or below | Separate positive/negative summaries |
| Maximum 50 | Limit to stay within token limits |

Each review is wrapped with `[REVIEW_START]` and `[REVIEW_END]` delimiters so the model can clearly identify individual review boundaries.

Based on actual Insadong data, out of 220 total reviews, there were 177 five-star reviews and 43 reviews with four stars or below.

---

## 3. Prompt Engineering — Iterative Improvement Process

Prompt engineering was the core of this project. Starting from a baseline, I improved the prompt over 4 stages, validating effectiveness through evaluation at each stage.

### 3-1. Baseline — Simple Instruction

```python
PROMPT_BASELINE = "Summarize the following accommodation reviews in 5 sentences or less:"
```

I started with the simplest possible prompt — just asking GPT-3.5-Turbo to summarize without any specific conditions.

**Result**: 10 wins out of 10 (vs. Yanolja's actual summary)

Surprisingly, even a simple prompt outperformed Yanolja's existing summaries. However, the output format was inconsistent, with mixed tone and style.

### 3-2. Enhancement 1 — Specifying Output Conditions

```python
prompt = """You are a summarization expert. Your goal is to summarize user accommodation reviews.

The summary must meet the following conditions:
1. All sentences must end in polite form.
2. Write in a tone that introduces the accommodation.
  2-1. Good examples
    a) Overall, reviewers say it was a good accommodation with decent soundproofing.
    b) There are reviews mentioning plans to revisit.
  2-2. Bad examples
    a) It was a good accommodation with decent soundproofing.
    b) I plan to revisit.
3. The summary should be between 2 and 5 sentences.

Please summarize the following accommodation reviews:"""
```

I added specific conditions including role assignment, polite tone, tone and manner guidelines, and sentence count limits. The key point was explaining the "introductory tone" through good and bad examples — guiding the model to write as a third party introducing the accommodation rather than copying reviews directly.

**Result**: 10 wins out of 10 (vs. Yanolja's actual summary)

### 3-3. Enhancement 2 — Improving Input Data Quality

Instead of the prompt, I improved the input data side. This is where the preprocessing logic (removing reviews under 30 characters, within 6 months, 50 review limit) was applied.

**Result**: 7 wins, 3 losses out of 10 (vs. Yanolja's actual summary)

Interestingly, performance actually dropped. The analysis showed that removing short reviews that contained key keywords ("best location," "clean") reduced the diversity of summaries. However, since noise removal is necessary in the long run, I kept the preprocessing and focused on compensating through prompt improvements.

### 3-4. Enhancement 3 — Few-Shot Prompting

To recover from the performance drop, I introduced Few-shot Prompting. I generated an "exemplary summary" of Pangyo location reviews using GPT-4-Turbo and provided it as an example.

```python
# Generate exemplary summary of Pangyo reviews using GPT-4-Turbo
reviews_1shot, _ = preprocess_reviews('./res/ninetree_pangyo.json')
summary_1shot = summarize(
    reviews_1shot, prompt, temperature=0.0, model='gpt-4-turbo-2024-04-09'
).choices[0].message.content

# 1-Shot prompt construction
prompt_1shot = f"""You are a summarization expert. Your goal is to summarize
user accommodation reviews. Here are example reviews and their summary.

Example reviews:
{reviews_1shot}
Example summary:
{summary_1shot}

Please summarize the following accommodation reviews:"""
```

The key idea is **using GPT-4's output as a guide for GPT-3.5**. The expensive GPT-4 is used only once for generating the example, while the cost-effective GPT-3.5 handles actual service requests.

**1-Shot Result**: 9 wins, 1 loss out of 10 (vs. Yanolja's actual summary)

I also tried extending to 2-Shot with Yongsan location reviews.

**2-Shot Result**: 9 wins, 1 loss out of 10 (vs. Yanolja's actual summary)

Since there was no performance difference between 1-Shot and 2-Shot, I ultimately adopted 1-Shot. If there's no quality improvement relative to token cost, using fewer examples is the rational choice.

### Prompt Improvement Summary

| Version | Strategy | W/L/D | Notes |
|---------|----------|-------|-------|
| Baseline | Simple instruction | 10/0/0 | Inconsistent format |
| v1 | Explicit conditions | 10/0/0 | Improved tone and manner |
| v2 | Enhanced preprocessing | 7/3/0 | Data quality↑, Diversity↓ |
| v3 (1-Shot) | Few-Shot | 9/1/0 | Using GPT-4 examples |
| v4 (2-Shot) | Few-Shot | 9/1/0 | No difference from 1-Shot |

---

## 4. Evaluation Method — MT-Bench Style Pairwise Evaluation

How do you evaluate the quality of LLM-generated summaries? Traditional metrics like BLEU and ROUGE don't adequately reflect the semantic quality of text. In this project, I adopted the **Pairwise Comparison** method proposed in the MT-Bench paper.

### Why Comparison Instead of Scoring?

- **Problem with scoring**: "How many points does this summary deserve?" → The criteria are vague (what does a 3-point summary even mean?)
- **Advantage of comparison**: "Which is better, A or B?" → Relative judgment is clearer and more consistent

### Evaluation Implementation

GPT-4o is used as a judge to compare two summaries.

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

### Evaluation Criteria

GPT-4o judges based on the following 6 criteria:

- **Helpfulness**: Is it practically helpful to the user?
- **Relevance**: Is it relevant to the review content?
- **Accuracy**: Is the summary factually grounded?
- **Depth**: Does it have sufficient depth?
- **Creativity**: Is it a comprehensive summary rather than a simple listing?
- **Level of Detail**: Does it include an appropriate level of detail?

### Large-Scale Evaluation — Using Temperature

To obtain diverse outputs from the same prompt, I generated 10 summaries with `temperature=1.0` and measured the win rate by comparing each against Yanolja's actual summaries.

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

## 5. Summary Generation

The final summary is generated by calling GPT-3.5-Turbo with `temperature=0.0` to ensure consistent results.

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

Positive and negative reviews are summarized separately, allowing users to grasp the pros and cons of an accommodation at a glance.

---

## 6. Gradio Demo

I built a simple web interface using Gradio.

```python
def run_demo():
    demo = gr.Interface(
        fn=fn,
        inputs=[gr.Radio(['Insadong', 'Pangyo', 'Yongsan'], label='Accommodation')],
        outputs=[
            gr.Textbox(label='High Rating Summary'),
            gr.Textbox(label='Low Rating Summary')
        ]
    )
    demo.launch(share=True)
```

When a user selects an accommodation, the crawled reviews are preprocessed and GPT generates separate positive/negative summaries. With `share=True`, an external sharing link can also be generated.

---

## Lessons Learned

**Data quality and prompts can have a trade-off relationship.**
Strengthening preprocessing reduces noise but also decreases information volume. Few-shot Prompting proved to be an effective compensating measure.

**Using GPT-4's output as a guide for GPT-3.5 is a cost-effective strategy.**
Generating a model answer once with GPT-4 and running the actual service with GPT-3.5 lets you achieve both quality and cost efficiency.

**Pairwise Comparison is intuitive for LLM evaluation.**
The comparison approach of "which is better, A or B?" was more consistent and easier to interpret than score-based evaluation.

**One example may be enough for Few-shot.**
There was no performance difference between 1-Shot and 2-Shot. Rather than blindly increasing examples, choosing one good example matters more.

---

## Tech Stack

| Category | Technology |
|----------|------------|
| Crawling | Selenium, BeautifulSoup |
| Summary Model | GPT-3.5-Turbo (service), GPT-4-Turbo (example generation) |
| Evaluation Model | GPT-4o |
| Prompting | Few-shot Prompting (1-Shot) |
| Demo | Gradio |
| Language | Python |
