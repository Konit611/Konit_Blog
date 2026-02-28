---
title: "Prompt Engineering: From Theory to Building Real Services (1)"
date: 2026-02-28 17:47
excerpt: A summary of core prompt engineering theories and lessons learned from optimizing gpt-3.5-turbo.
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

## 1. Prompt Engineering: The Art of Design

I started studying AI Engineering to efficiently build my own AI services. As the first entry, I'd like to document the core theories of prompt engineering and the trial-and-error process of optimizing the cost-effective model `gpt-3.5-turbo`.

Prompt engineering is not simply about asking better questions — it is **a design discipline for guiding model outputs to align with business logic**.

### Service Design Workflow

1. **Define Problem Constraints**: Clearly define the scope of the task the model needs to solve.
2. **Set Evaluation Criteria**: Establish metrics (MMLU, G-Eval, etc.) to measure what constitutes a correct answer.
3. **Select the Model**: Choose the optimal model considering the balance between performance and **cost**.

---

## 2. The Key to Cost and Performance: English Prompts and Tokens

In the productionization phase, **cost optimization** is paramount. Models understand text in units called tokens, and this is where the critical reason for using English prompts emerges.

### Why Write Prompts in English?

- **Token Efficiency**: Most LLM tokenizers are optimized based on English datasets. Korean requires multiple tokens to represent a single character, whereas English is efficiently encoded at the word level. (This directly reduces API costs)
- **Reasoning Performance**: The overwhelming majority of model training data is in English. Therefore, when issuing complex logical instructions, the model more accurately understands the intent when prompted in English.

### Actual API Call Structure

An efficient approach is to write instructions (System/User) in English while passing data in the target language.

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

## 3. Case Study: A Logic Problem That Stumped gpt-3.5-turbo

The cost-effective `gpt-3.5-turbo` often shows limitations in complex logical reasoning. Let's examine the improvement process through an actual MMLU (Massive Multitask Language Understanding) style problem.

### First Attempt: Simple Zero-Shot (Failure)

We attempted an MMLU-style math problem with a simple Zero-Shot approach.

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

- **Result**: The model selected **(C) 8**.
- **Cause of Failure**: It failed to accurately find the vertices of the system of inequalities and derived an incorrect answer through simple reasoning.

### Improvement: Role Assignment + CoT (Chain of Thought)

After analyzing why the simple Zero-Shot failed, we applied two prompt techniques.

1. **Role Assignment**: Assigned the role "You are a Professional in Mathematics." at the system level to induce expert-level mathematical reasoning.
2. **CoT (Chain of Thought)**: Explicitly stated "Think carefully" and "Give reasons" to enforce step-by-step thinking.

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

- **Result**: The model arrived at the correct answer **(B) 6**.

The actual Reasoning output from the model is as follows:

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

The interesting point is that even though the model initially produced an incorrect result of M-m=0, it **self-corrected and arrived at the correct answer**. Thanks to CoT forcing the model to "explain its reasoning," self-correction became possible — something that was impossible with simple Zero-Shot.

**Summary of Applied Techniques:**

| Technique | First Attempt | After Improvement |
|---|---|---|
| Role Assignment | None | Mathematics expert role |
| Reasoning Guidance | Simple answer request | Forced explanation (CoT) |
| Result | **(C) 8** (Wrong) | **(B) 6** (Correct) |

---

## 4. Limitations and Evaluation of Prompt Engineering

Prompt engineering is powerful, but it is not magic.

- **Strengths**: Improves performance and controls costs without retraining.
- **Limitations**: There are constraints from the model's knowledge boundaries and input length (Context Window).
- **Evaluation Metrics**: Prompt quality must be objectively verified through metrics such as MMLU (knowledge), MT Bench (conversation), and ROUGE (summarization).

---

## 5. Coming Next: Building a Jalan Review Analysis Service

Through theory and foundational testing, we confirmed the optimization potential of `gpt-3.5-turbo`. In the next post, we will serialize a hands-on demo service development story — using review data from an actual accommodation platform to have AI automatically extract insights on cleanliness, service quality, and value for money.

Stay tuned for the real-world AI Engineering process that minimizes cost while maximizing performance!
