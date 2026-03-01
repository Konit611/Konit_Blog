---
title: "提示工程：从理论到实战服务构建（2）"
date: 2026-03-01 13:54
excerpt: 我构建了一个爬取Yanolja住宿评论并使用GPT-3.5-Turbo自动摘要的流水线，通过Few-shot Prompting和Pairwise Evaluation提升了摘要质量。
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

# 使用GPT构建Yanolja住宿评论自动摘要服务

预订住宿时逐一阅读数百条评论是一件相当繁琐的事。从"帮我总结一下这个住宿的评论"这个简单的想法出发，我构建了一个爬取Yanolja住宿评论并使用GPT自动摘要的流水线，以下是整个过程的总结。

---

## 整体流水线概览

```
[爬虫] → [预处理] → [提示工程] → [摘要生成] → [评估] → [Demo部署]
```

1. **爬虫**：使用Selenium + BeautifulSoup采集Yanolja评论
2. **预处理**：日期/长度过滤，分离正面和负面评论
3. **摘要**：GPT-3.5-Turbo + Few-shot Prompting
4. **评估**：MT-Bench风格的Pairwise Evaluation（GPT-4o）
5. **Demo**：Gradio Web界面

---

## 1. 数据采集 — 爬取Yanolja评论

Yanolja的评论页面在滚动时动态加载评论，因此仅靠简单的HTTP请求无法获取数据。我使用Selenium控制浏览器反复滚动，然后用BeautifulSoup解析HTML。

```python
def crawl_yanolja_reviews(name, url):
    review_list = []
    driver = webdriver.Chrome()
    driver.get(url)

    time.sleep(3)

    # 反复滚动以加载所有动态加载的评论
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

采集到的数据格式如下：

```json
[
    {"review": "干净又很棒", "stars": 5, "date": "22小时前"},
    {"review": "位置很好，住宿也干净~", "stars": 5, "date": "2天前"},
    ...
]
```

星级评分的计数逻辑比较独特。由于Yanolja使用SVG `<path>`元素渲染星星，我通过统计具有`fill="currentColor"`属性但没有`fill-rule`属性的path元素数量来提取星级评分。

我采集了仁寺洞、板桥、龙山三个地点的评论用于实验。

---

## 2. 数据预处理

将原始评论直接输入模型会产生大量噪声，导致摘要质量下降。我按照以下标准进行了预处理：

```python
def preprocess_reviews(path='./res/reviews.json'):
    with open(path, 'r', encoding='utf-8') as f:
        review_list = json.load(f)

    reviews_good, reviews_bad = [], []

    current_date = datetime.datetime.now()
    date_boundary = current_date - datetime.timedelta(days=6*30)

    for r in review_list:
        review_date = parser.parse(r['date'])

        # 1) 仅使用最近6个月的评论
        if review_date < date_boundary:
            continue
        # 2) 移除少于30个字符的短评论
        if len(r['review']) < 30:
            continue

        # 3) 根据星级分离正面/负面评论
        if r['stars'] == 5:
            reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
        else:
            reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')

    # 4) 最多限制50条
    reviews_good = reviews_good[:50]
    reviews_bad = reviews_bad[:50]

    return '\n'.join(reviews_good), '\n'.join(reviews_bad)
```

### 预处理标准

| 标准 | 原因 |
|------|------|
| 最近6个月内 | 旧评论无法反映住宿的当前状况 |
| 30个字符以上 | "不错"、"干净"等短评论对摘要没有帮助 |
| 5星 vs 4星及以下 | 为了分别生成正面和负面摘要 |
| 最多50条 | 在Token限制内进行处理 |

每条评论都用`[REVIEW_START]`和`[REVIEW_END]`分隔符包裹，使模型能够清楚地识别各条评论的边界。

基于仁寺洞的实际数据，在总共220条评论中，5星评论177条，4星及以下评论43条。

---

## 3. 提示工程 — 迭代改进过程

提示工程是这个项目的核心。从Baseline开始，我经过4个阶段改进了提示，并在每个阶段通过评估验证效果。

### 3-1. Baseline — 简单指令

```python
PROMPT_BASELINE = "请将以下住宿评论总结为5句话以内："
```

从最简单的提示开始，不附加任何条件地要求GPT-3.5-Turbo进行摘要。

**结果**：10战10胜（vs. Yanolja实际摘要）

出乎意料的是，即使是简单的提示也优于Yanolja现有的摘要。但输出格式不一致，语气混乱。

### 3-2. 改进1 — 明确输出条件

```python
prompt = """你是一位摘要专家。当给定用户住宿评论时，你的目标是进行摘要。

摘要结果必须满足以下条件：
1. 所有句子必须使用礼貌体结尾。
2. 请用介绍住宿的语气撰写。
  2-1. 好的示例
    a) 总体来说，评论表示这是一家不错的住宿，隔音效果也不错。
    b) 有评论提到计划再次入住。
  2-2. 不好的示例
    a) 这是一家不错的住宿，隔音效果也不错。
    b) 我计划再次入住。
3. 摘要结果请控制在2到5句话之间。

请对以下住宿评论进行摘要："""
```

添加了角色分配、礼貌用语、语气风格、句数限制等具体条件。关键在于通过好的示例和不好的示例来说明"介绍性语气"——引导模型不是直接搬运评论，而是以第三方介绍住宿的文体来写。

**结果**：10战10胜（vs. Yanolja实际摘要）

### 3-3. 改进2 — 提高输入数据质量

这次改进的不是提示，而是输入数据。前面介绍的预处理逻辑（移除30字以下、6个月内、50条限制）就是在这个阶段应用的。

**结果**：10战7胜3负（vs. Yanolja实际摘要）

有趣的是，性能反而下降了。分析表明，移除了包含关键词的短评论（"位置最佳"、"很干净"），导致摘要的多样性降低。但从长远来看，去噪是必要的，因此保留预处理并从提示方面进行补偿。

### 3-4. 改进3 — Few-Shot Prompting

为了弥补性能下降，我引入了Few-shot Prompting。使用GPT-4-Turbo生成板桥地点评论的"模范摘要"，并将其作为示例一起提供。

```python
# 使用GPT-4-Turbo生成板桥评论的模范摘要
reviews_1shot, _ = preprocess_reviews('./res/ninetree_pangyo.json')
summary_1shot = summarize(
    reviews_1shot, prompt, temperature=0.0, model='gpt-4-turbo-2024-04-09'
).choices[0].message.content

# 1-Shot提示构成
prompt_1shot = f"""你是一位摘要专家。当给定用户住宿评论时，
你的目标是进行摘要。以下是评论和摘要的示例。

示例评论：
{reviews_1shot}
示例摘要结果：
{summary_1shot}

请对以下住宿评论进行摘要："""
```

核心思路是**将GPT-4的输出作为GPT-3.5的指导**。昂贵的GPT-4仅用于生成示例一次，实际服务中使用性价比更高的GPT-3.5。

**1-Shot结果**：10战9胜1负（vs. Yanolja实际摘要）

我还尝试用龙山地点的评论扩展到2-Shot。

**2-Shot结果**：10战9胜1负（vs. Yanolja实际摘要）

由于1-Shot和2-Shot没有性能差异，最终采用了1-Shot。如果相对于Token成本没有质量提升，使用更少的示例是合理的选择。

### 提示改进过程总结

| 版本 | 策略 | 胜/负/平 | 备注 |
|------|------|----------|------|
| Baseline | 简单指令 | 10/0/0 | 格式不一致 |
| v1 | 明确条件 | 10/0/0 | 改善语气风格 |
| v2 | 强化预处理 | 7/3/0 | 数据质量↑，多样性↓ |
| v3（1-Shot） | Few-Shot | 9/1/0 | 利用GPT-4示例 |
| v4（2-Shot） | Few-Shot | 9/1/0 | 与1-Shot无差异 |

---

## 4. 评估方法 — MT-Bench风格Pairwise Evaluation

如何评估LLM生成的摘要质量？BLEU、ROUGE等传统指标无法充分反映文本的语义质量。本项目采用了MT-Bench论文提出的**Pairwise Comparison**方法。

### 为什么选择比较而非评分？

- **评分方式的问题**："这个摘要值几分？" → 标准模糊（3分的摘要是什么样的？）
- **比较方式的优势**："A和B哪个更好？" → 相对判断更清晰、更一致

### 评估实现

使用GPT-4o作为裁判（Judge）来比较两个摘要。

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

### 评估标准

GPT-4o基于以下6个标准进行判断：

- **Helpfulness（有用性）**：对用户是否有实际帮助
- **Relevance（相关性）**：是否与评论内容相关
- **Accuracy（准确性）**：摘要是否基于事实
- **Depth（深度）**：是否具有足够的深度
- **Creativity（创造性）**：是否是综合性摘要而非简单罗列
- **Level of Detail（细节程度）**：是否包含适当的细节

### 大规模评估 — 利用temperature

为了从相同的提示获得多样化的输出，我使用`temperature=1.0`生成10个摘要，分别与Yanolja的实际摘要进行比较，测量胜率。

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

## 5. 摘要生成

最终摘要通过`temperature=0.0`调用GPT-3.5-Turbo来确保结果的一致性。

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

正面评论和负面评论分别进行摘要，让用户能一目了然地了解住宿的优缺点。

---

## 6. Gradio Demo

使用Gradio构建了简单的Web界面。

```python
def run_demo():
    demo = gr.Interface(
        fn=fn,
        inputs=[gr.Radio(['仁寺洞', '板桥', '龙山'], label='住宿')],
        outputs=[
            gr.Textbox(label='高评分摘要'),
            gr.Textbox(label='低评分摘要')
        ]
    )
    demo.launch(share=True)
```

用户选择住宿后，系统对爬取的评论进行预处理，GPT分别生成正面/负面摘要并展示。通过`share=True`还可以生成外部分享链接。

---

## 经验总结

**数据质量和提示之间可能存在权衡关系。**
强化预处理可以减少噪声，但也会减少信息量。此时Few-shot Prompting成为了有效的补偿手段。

**将GPT-4的输出作为GPT-3.5的指导是一种高性价比策略。**
用GPT-4生成一次模范答案，实际服务中使用GPT-3.5运营，可以同时兼顾质量和成本。

**Pairwise Comparison对于LLM评估来说很直观。**
"A和B哪个更好"的比较方式比基于评分的评估更一致，也更容易解读。

**Few-shot有时一个示例就够了。**
1-Shot和2-Shot之间没有性能差异。与其盲目增加示例数量，不如精心挑选一个好的示例更为重要。

---

## 技术栈

| 类别 | 技术 |
|------|------|
| 爬虫 | Selenium、BeautifulSoup |
| 摘要模型 | GPT-3.5-Turbo（服务）、GPT-4-Turbo（示例生成） |
| 评估模型 | GPT-4o |
| 提示策略 | Few-shot Prompting（1-Shot） |
| Demo | Gradio |
| 编程语言 | Python |
