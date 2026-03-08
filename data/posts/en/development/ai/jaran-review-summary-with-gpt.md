---
title: "Prompt Engineering: From Theory to Building a Production Service (3)"
date: 2026-03-08 13:00
excerpt: Summarizing Jalan's Japanese accommodation reviews with GPT, and how we leveraged GPT-4's output as a one-shot example to elevate GPT-3.5's quality.
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

# Summarizing Accommodation Reviews with LLMs - Model Comparison and Prompt Engineering

## Introduction

Accommodation platforms accumulate a massive number of reviews. Summarizing these reviews for quick comprehension would greatly improve the user experience. In this post, we share the process of summarizing Japanese accommodation reviews using OpenAI's GPT models and **elevating the quality of a cheaper model through prompt engineering alone**.

The key flow is as follows:

> Simple prompt → GPT-3.5 (4 points), GPT-4 (5 points)
> → Detailed prompt applied → GPT-3.5 (dropped to 2 points)
> → Using GPT-4 output as a one-shot example → GPT-3.5 (achieved 5 points)

## 1. Crawling Jalan Reviews

To obtain review data for summarization, we crawled reviews from the Japanese accommodation booking platform Jalan (jalan.net). We used Selenium to render the pages and BeautifulSoup to parse them.

```python
from bs4 import BeautifulSoup
from selenium import webdriver

driver = webdriver.Chrome()
driver.get(url)

soup = BeautifulSoup(driver.page_source, 'html.parser')
containers = soup.select('.jlnpc-kuchikomiCassette')
```

From each review cassette (`.jlnpc-kuchikomiCassette`), we extract the following information:

- **Review body** (`.jlnpc-kuchikomiCassette__postBody`)
- **Overall score** (`.jlnpc-kuchikomiCassette__totalRate`)
- **Stay date** (format: "2025年12月")
- **Detailed ratings** - Room, Bath, Cuisine, Service, Cleanliness

Detailed ratings are extracted from a rating list composed of `dt/dd` pairs, with a regex fallback extraction from text if this method fails.

```python
def extract_ratings(container):
    ratings = {}
    rate_list = container.select_one('.jlnpc-kuchikomiCassette__rateList')
    if rate_list:
        dts = rate_list.find_all('dt')
        for dt in dts:
            label = dt.get_text(strip=True)
            dd = dt.find_next_sibling('dd')
            # Map Japanese labels to English field names via RATING_MAP
            ...
    return ratings
```

Pagination is handled by clicking the "Next" button, traversing up to 5 pages. Reviews from 3 accommodations were saved as individual JSON files.

## 2. Review Data Preprocessing

First, we preprocess the review data in JSON format. We filter only reviews from the last 6 months and classify them into 5-star (positive) and others (negative). Each review is wrapped with `[REVIEW_START]` and `[REVIEW_END]` tags so the LLM can clearly recognize the boundaries of each review.

```python
for r in review_list:
    # Filter for last 6 months
    if review_date < date_boundary:
        continue

    if r['star'] == 5:
        reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
    else:
        reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
```

## 3. Comparing Models with a Baseline Prompt

We started with the simplest possible prompt.

```
Please summarize the following accommodation reviews in about 5 lines:
```

### GPT-3.5's Results

GPT-3.5 summarized by numbering and listing individual reviews. While it did condense the content of each review, **the result was closer to review excerpts than an integrated summary**.

### GPT-4's Results

GPT-4, on the other hand, consolidated information by theme. It organized the overall review trends into a single summary, categorized by meals, spa, staff service, rooms, and congestion.

### Quality Evaluation with G-Eval

To quantitatively compare the quality of summaries, we applied the evaluation method from the [G-Eval](https://arxiv.org/abs/2303.16634) paper. We evaluated the Coherence metric on a 1-5 scale.

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

| Model | Coherence Score | Evaluation Summary |
|------|:-:|------|
| GPT-3.5 | 4 | Organized by number, but a list of individual mini-summaries rather than one integrated summary |
| GPT-4 | 5 | Well-structured by theme with a consistent flow |

GPT-4 clearly showed better results, but the cost difference is significant. Is there a way to improve GPT-3.5's quality?

---

## 4. Detailed Prompt - Backfired

We tried adding specific conditions to the prompt.

```
You are a summarization expert.

The summary must meet the following conditions:
1. All sentences must end in polite/formal language
2. Write in a tone that introduces the accommodation
3. The summary should be between 2 and 5 sentences
```

We even provided good and bad examples to clearly specify the tone and manner. However, the results were unexpected.

**GPT-3.5's Coherence Score: 2 points**

GPT-3.5 failed to follow the instructions properly. Instead of summarizing, it nearly copied the original reviews verbatim, and awkward expressions like "it was lovely it is" and "I thought it is" appeared at the end of sentences. As the conditions became more complex, even the basic summarization ability deteriorated.

This provides an important lesson: **Complex instructions that exceed the model's capabilities can backfire.**

---

## 5. One-Shot Prompt (1st Attempt) - The Same-Data Trap

Here, the key idea emerges. Generate a good summary example with GPT-4 first, then include it as a one-shot example in GPT-3.5's prompt.

```python
# Step 1: Generate example summary with GPT-4
summary_1shot = summarize(reviews, PROMPT_ONE_SHOT,
                          model='gpt-4-turbo-2024-04-09')

# Step 2: Include the generated example in GPT-3.5's prompt
prompt_1shot = f"""You are a summarization expert.
...
Below is an example of reviews and their summary.
Example reviews:
{reviews_1shot}
Example summary:
{summary_1shot}

Please summarize the following accommodation reviews:"""

# Step 3: Run with GPT-3.5
result = summarize(reviews, prompt_1shot,
                   temperature=1.0, model='gpt-3.5-turbo-0125')
```

### Result: Coherence 5 Points, But There's a Problem

The G-Eval Coherence score reached 5 points, but looking at the actual output reveals the issue:

> The view from the Japanese-style lakeside room was wonderful, and in the immaculately clean space, the dinner and breakfast buffets were very satisfying. The children were especially delighted with the rooftop spa, and we were able to spend precious family time together. The staff were warm and it was a great trip.

This result reads **like an actual review being written, not a review summary**. "~was very satisfying" and "~it was a great trip" are a guest's personal impressions, not the summary tone of "~is highly rated" or "~many guests note that".

The cause was that **the one-shot example reviews and the actual target reviews were the same data**. Since both `reviews_1shot` and `reviews` came from the same `reviews.json`, the model didn't learn the tone from the example but simply repeated the review content it had already seen.

---

## 6. One-Shot Prompt (2nd Attempt) - Solved with Separate Examples

To solve the problem, we made two changes:

1. **Hard-coded the one-shot example with a separate review/summary pair** - Used completely different data from the summarization target
2. **Specified the summary tone and structure in the prompt** - Defined the "~is highly rated" pattern and summary order

```python
# Separate example data different from the target
example_reviews = """[REVIEW_START]The room was spacious and comfortable. The soundproofing was solid...[REVIEW_END]
[REVIEW_START]It was close to the station with good access. The front desk staff were friendly...[REVIEW_END]
..."""

# Directly wrote a summary example in the desired tone
example_summary = """Many reviews praise the rooms as spacious, clean, and well-soundproofed.
The convenient station access and friendly front desk service are frequently mentioned...."""

prompt_1shot = f"""You are a summarization expert.
...
4. Rather than writing individual reviews as-is, synthesize trends from
   multiple reviews using phrases like "is highly rated" or "many guests note that".
5. Structure the summary in this order:
   "Facilities/Rooms → Meals → Service/Staff → Overall Evaluation".

Below is an example of reviews and their summary.
Example reviews:
{example_reviews}
Example summary:
{example_summary}

Please summarize the following accommodation reviews:"""
```

### Result: Coherence 5 Points Even at temperature=1.0

Even with the high randomness of `temperature=1.0`, the model consistently achieved 5 points. The separate examples and clear structural instructions proved effective.

The key difference lies in the output tone. In the 1st attempt, the output read "was very satisfying" (personal experience), while in the 2nd attempt, it changed to "many guests praise" (review trend summary).

Final summary result:

> Overall, the accommodation is highly rated as excellent. The rooms are clean with great views, making it ideal for families and groups of friends. Both dinner and breakfast offer a rich variety of dishes, and the sky spa in particular receives very high satisfaction. The front desk and staff service are also praised, with many guests noting it made for a comfortable trip. Many express a desire to revisit.

---

## Summary

| Approach | Model | Coherence | Notes |
|------|------|:-:|------|
| Baseline prompt | GPT-3.5 | 4 | Individual review listing |
| Baseline prompt | GPT-4 | 5 | Thematic integrated summary |
| Detailed conditions prompt | GPT-3.5 | 2 | Quality drop from instruction overload |
| One-shot (same data example) | GPT-3.5 | 5 | High score but output in review tone |
| One-shot (separate data example) | GPT-3.5 | 5 | Correctly output in summary tone |

The key takeaways from this experiment are:

1. **Good examples beat complex instructions** - Rather than listing conditions that exceed the model's capabilities, showing one example of the desired result is far more effective.
2. **One-shot examples and actual input must use different data** - Using the same data prevents the model from learning the pattern and causes it to repeat the content verbatim. The example teaches "what tone to summarize in," not a signal to copy content.
3. **Quantitative evaluation alone is insufficient** - Even with a Coherence score of 5, the actual output tone may differ from the intent. G-Eval evaluates structural coherence, but tone appropriateness must be verified separately.
4. **Automating evaluation is crucial** - Introducing LLM-based evaluation like G-Eval enables quantitative comparison of prompt changes, allowing improvement without relying on intuition.
