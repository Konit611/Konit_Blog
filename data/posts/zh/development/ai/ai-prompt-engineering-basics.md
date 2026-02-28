---
title: "提示词工程：从理论到实战服务构建（1）"
date: 2026-02-28 17:47
excerpt: 总结了在优化gpt-3.5-turbo过程中积累的提示词工程核心理论与经验教训。
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

## 1. 提示词工程：设计的美学

为了高效构建属于自己的AI服务，我开始了AI Engineering的学习。作为第一篇记录，我想整理一下提示词工程的核心理论，以及在优化高性价比模型`gpt-3.5-turbo`过程中的试错经验。

提示词工程不仅仅是学会如何提问，而是**引导模型输出符合业务逻辑的设计技术**。

### 服务设计工作流

1. **定义问题条件**：明确定义模型需要解决的Task范围。
2. **设定评估标准**：建立衡量什么是正确答案的指标（MMLU、G-Eval等）。
3. **选择模型**：在性能与**成本（Cost）** 之间取得平衡，选择最优模型。

---

## 2. 成本与性能的关键：英文提示词与Token

在服务化阶段，最重要的是**成本优化**。模型以Token为单位理解文本，这里就出现了必须使用英文提示词的关键原因。

### 为什么要用英文编写？

- **Token效率**：大多数LLM的Tokenizer基于英语数据集进行了优化。中文、韩文等表示一个字符需要消耗多个Token，而英语可以按单词高效编码。（最终降低API成本）
- **推理性能**：模型训练数据中英语占据压倒性比例。因此，在下达复杂逻辑指令时，使用英语指示能让模型更准确地理解意图。

### 实际API调用结构

指令（System/User）用英文编写，数据用目标语言传递的方式最为高效。

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

## 3. Case Study：让gpt-3.5-turbo困惑的逻辑问题

高性价比模型`gpt-3.5-turbo`在复合逻辑推理中经常表现出局限性。让我们通过一道实际的MMLU（大规模多任务语言理解）格式的题目来查看改进过程。

### 第一次尝试：简单Zero-Shot（失败）

用简单的Zero-Shot方式尝试了一道MMLU风格的数学题。

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

- **结果**：模型选择了**(C) 8**。
- **失败原因**：未能准确求出联立不等式的顶点，通过简单推理得出了错误答案。

### 改进：角色赋予 + CoT（Chain of Thought）应用

分析了简单Zero-Shot失败的原因后，应用了两种提示词技术。

1. **角色赋予**：在System层面指定"You are a Professional in Mathematics."的角色，引导数学专家级推理。
2. **CoT（Chain of Thought）**：明确写出"Think carefully"和"Give reasons"，强制进行逐步思考。

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

- **结果**：模型得出了正确答案**(B) 6**。

模型的实际Reasoning输出如下：

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

有趣的是，模型在中途得出了M-m=0的错误结果，但却**自行重新审视并最终到达了正确答案**。正是因为通过CoT强制要求"解释理由"，才使得简单Zero-Shot中不可能实现的自我纠正（Self-Correction）成为可能。

**应用的核心技术总结：**

| 技术 | 第一次尝试 | 改进后 |
|---|---|---|
| 角色赋予 | 无 | 数学专家角色指定 |
| 推理引导 | 简单回答请求 | 强制解释理由（CoT） |
| 结果 | **(C) 8**（错误） | **(B) 6**（正确） |

---

## 4. 提示词工程的局限与评估

提示词工程虽然强大，但并非万能。

- **优点**：无需重新训练即可改善性能并控制成本。
- **局限**：存在模型知识边界和输入长度（Context Window）的限制。
- **评估指标**：需要通过MMLU（知识）、MT Bench（对话）、ROUGE（摘要）等指标客观验证提示词的质量。

---

## 5. 下期预告：构建Jalan评论分析服务

通过理论和基础测试，我们确认了`gpt-3.5-turbo`的优化潜力。在下一篇文章中，我们将连载一个实战项目——利用实际住宿平台的评论数据，让AI自动提取清洁度、服务质量和性价比等洞察的演示服务开发记。

敬请期待降低成本同时最大化性能的AI Engineering实战流程！
