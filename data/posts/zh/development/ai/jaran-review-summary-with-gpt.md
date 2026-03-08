---
title: "提示工程：从理论到实战服务构建（3）"
date: 2026-03-08 13:00
excerpt: 使用GPT对Jalan的日语住宿评论进行摘要，并利用GPT-4的输出作为One-shot示例来提升GPT-3.5的质量。
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

# 利用LLM进行住宿评论摘要 - 模型比较与提示工程

## 前言

住宿平台上积累了大量评论。如果能将这些评论进行摘要以便一目了然，将大大改善用户体验。本文分享了使用OpenAI的GPT模型对日语住宿评论进行摘要，并**仅通过提示工程就提升了低成本模型质量的过程**。

核心流程如下：

> 简单提示 → GPT-3.5（4分）、GPT-4（5分）
> → 应用详细提示 → GPT-3.5（降至2分）
> → 利用GPT-4输出作为One-shot示例 → GPT-3.5（达到5分）

## 1. Jalan评论爬取

为了获取用于摘要的评论数据，我们从日本住宿预订平台Jalan（jalan.net）爬取了评论。使用Selenium渲染页面，再用BeautifulSoup进行解析。

```python
from bs4 import BeautifulSoup
from selenium import webdriver

driver = webdriver.Chrome()
driver.get(url)

soup = BeautifulSoup(driver.page_source, 'html.parser')
containers = soup.select('.jlnpc-kuchikomiCassette')
```

从每个评论卡片（`.jlnpc-kuchikomiCassette`）中提取以下信息：

- **评论正文**（`.jlnpc-kuchikomiCassette__postBody`）
- **总评分**（`.jlnpc-kuchikomiCassette__totalRate`）
- **入住日期**（"2025年12月"格式）
- **细项评分** - 房间、浴室、餐饮、服务、清洁度

细项评分从由`dt/dd`对组成的评分列表中提取，如果该方法失败，则通过正则表达式从文本中回退提取。

```python
def extract_ratings(container):
    ratings = {}
    rate_list = container.select_one('.jlnpc-kuchikomiCassette__rateList')
    if rate_list:
        dts = rate_list.find_all('dt')
        for dt in dts:
            label = dt.get_text(strip=True)
            dd = dt.find_next_sibling('dd')
            # 通过RATING_MAP将日语标签映射为英文字段名
            ...
    return ratings
```

分页通过点击"下一页"按钮实现，最多遍历5页。3个住宿设施的评论分别保存为JSON文件。

## 2. 评论数据预处理

首先，对JSON格式的评论数据进行预处理。仅筛选最近6个月内的评论，并将其分为5星（正面）和其他（负面）。每条评论用`[REVIEW_START]`和`[REVIEW_END]`标签包裹，以便LLM能够清楚地识别每条评论的边界。

```python
for r in review_list:
    # 筛选最近6个月
    if review_date < date_boundary:
        continue

    if r['star'] == 5:
        reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
    else:
        reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
```

## 3. 使用基线提示进行模型比较

从最简单的提示开始。

```
请将以下住宿设施的评论整理为约5行的摘要：
```

### GPT-3.5的结果

GPT-3.5通过编号列举的方式对各条评论进行了摘要。虽然确实压缩了每条评论的内容，但**与其说是综合摘要，不如说更接近评论摘录**。

### GPT-4的结果

相比之下，GPT-4按主题整合了信息。它将整体评论趋势按餐饮、温泉、员工服务、客房、拥挤程度等类别进行分类，整理成了一份统一的摘要。

### 使用G-Eval进行质量评估

为了定量比较摘要质量，我们应用了[G-Eval](https://arxiv.org/abs/2303.16634)论文的评估方法。使用1-5分的量表评估Coherence（连贯性）指标。

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

| 模型 | Coherence得分 | 评估摘要 |
|------|:-:|------|
| GPT-3.5 | 4 | 按编号整理，但不是一份综合摘要，而是单独迷你摘要的罗列 |
| GPT-4 | 5 | 按主题良好结构化，具有连贯的流程 |

GPT-4明显展示了更好的结果，但成本差异很大。有没有办法提升GPT-3.5的质量？

---

## 4. 详细提示 - 反而适得其反

我们尝试在提示中添加具体条件。

```
你是一位摘要专家。

摘要结果必须满足以下条件：
1. 所有句子必须以礼貌用语结尾
2. 以介绍住宿设施的语气撰写
3. 摘要结果应在2至5句之间
```

我们甚至提供了好的和坏的示例来明确指定语气风格。然而，结果出乎意料。

**GPT-3.5的Coherence得分：2分**

GPT-3.5未能正确遵循指示。它不是进行摘要，而是几乎原封不动地复制了原文评论，并且在句末出现了"觉得很棒了是的"、"认为了是的"等生硬的表达。随着条件变得复杂，基本的摘要能力反而下降了。

这给了我们一个重要的教训：**超出模型能力的复杂指示可能会适得其反。**

---

## 5. One-shot提示（第1次尝试） - 相同数据的陷阱

这里出现了关键想法。先用GPT-4生成一个好的摘要示例，然后将其作为one-shot示例放入GPT-3.5的提示中。

```python
# 第1步：用GPT-4生成示例摘要
summary_1shot = summarize(reviews, PROMPT_ONE_SHOT,
                          model='gpt-4-turbo-2024-04-09')

# 第2步：将生成的示例包含在GPT-3.5的提示中
prompt_1shot = f"""你是一位摘要专家。
...
以下是评论和摘要的示例。
示例评论：
{reviews_1shot}
示例摘要结果：
{summary_1shot}

请对以下住宿设施的评论进行摘要："""

# 第3步：用GPT-3.5执行
result = summarize(reviews, prompt_1shot,
                   temperature=1.0, model='gpt-3.5-turbo-0125')
```

### 结果：Coherence 5分，但存在问题

G-Eval Coherence得分达到了5分，但查看实际输出就会发现问题：

> 日式湖畔房间的景色非常美丽，在充满清洁感的空间里，晚餐和早餐自助餐非常令人满意。孩子们对屋顶水疗特别满意，度过了宝贵的家庭时光。工作人员的服务也很温暖，这是一次很好的旅行。

这个结果**不像是评论摘要，而像是在写实际评论**。"~非常满意"、"~是一次很好的旅行"是住客本人的感受，而不是"~获得了高度评价"、"~很多客人表示"这样的摘要语气。

原因是**one-shot示例的评论和实际摘要目标评论是相同的数据**。由于`reviews_1shot`和`reviews`都来自同一个`reviews.json`，模型没有从示例中学习语气，而是直接重复了它已经看过的评论内容。

---

## 6. One-shot提示（第2次尝试） - 用独立示例解决

为了解决这个问题，我们做了两个改变：

1. **将one-shot示例硬编码为独立的评论/摘要对** - 使用与摘要目标完全不同的数据
2. **在提示中明确摘要语气和结构** - 指定"~获得了高度评价"的模式和摘要顺序

```python
# 与摘要目标不同的独立示例数据
example_reviews = """[REVIEW_START]房间宽敞舒适。隔音也很好...[REVIEW_END]
[REVIEW_START]离车站近，交通便利。前台工作人员也很友好...[REVIEW_END]
..."""

# 直接编写期望语气的摘要示例
example_summary = """许多评论称赞房间宽敞整洁，隔音效果良好。
交通便利、前台服务友好也是经常被提及的优点...."""

prompt_1shot = f"""你是一位摘要专家。
...
4. 不要直接照搬单条评论，而是综合多条评论的趋势，
   使用"~获得了高度评价"、"~很多客人表示"等表达方式。
5. 摘要按照"设施·客房 → 餐饮 → 服务·员工 → 综合评价"
   的顺序构成。

以下是评论和摘要的示例。
示例评论：
{example_reviews}
示例摘要结果：
{example_summary}

请对以下住宿设施的评论进行摘要："""
```

### 结果：在temperature=1.0下也达到Coherence 5分

即使在`temperature=1.0`的高随机性下，也稳定地达到了5分。独立示例和明确的结构指示发挥了作用。

核心差异在于输出的语气。第1次尝试中出现的"非常满意"（亲身体验的感受），在第2次尝试中变为了"~获得了高度评价"（评论趋势摘要）。

最终摘要结果：

> 总体上获得了优秀住宿设施的高度评价。客房整洁、景色优美，非常适合家庭和朋友一起入住。晚餐和早餐的菜品丰富，尤其是天空水疗的满意度非常高。此外，前台和工作人员的服务也受到好评，很多客人表示度过了舒适的旅行。不少客人表达了再次入住的意愿。

---

## 总结

| 方式 | 模型 | Coherence | 备注 |
|------|------|:-:|------|
| 基线提示 | GPT-3.5 | 4 | 单独评论罗列 |
| 基线提示 | GPT-4 | 5 | 按主题综合摘要 |
| 详细条件提示 | GPT-3.5 | 2 | 指示过载导致质量下降 |
| One-shot（相同数据示例） | GPT-3.5 | 5 | 分数高但以评论语气输出 |
| One-shot（独立数据示例） | GPT-3.5 | 5 | 以摘要语气正确输出 |

本次实验获得的启示如下：

1. **好的示例胜过复杂的指示** - 与其罗列超出模型能力的条件，不如展示一个期望结果的示例，效果要好得多。
2. **one-shot示例和实际输入必须使用不同的数据** - 使用相同数据会导致模型无法学习模式，直接重复内容。示例是教"用什么语气进行摘要"，而不是复制内容的信号。
3. **仅靠定量评估是不够的** - 即使Coherence得分为5分，实际输出的语气也可能与预期不同。G-Eval评估的是结构连贯性，但语气的适当性需要另外验证。
4. **评估自动化很重要** - 引入G-Eval等基于LLM的评估，可以定量比较提示变更的效果，实现不依赖直觉的改进。
