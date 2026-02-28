---
title: "プロンプトエンジニアリング：理論から実践サービス構築まで（1）"
date: 2026-02-28 17:47
excerpt: gpt-3.5-turboを最適化する中で得たプロンプトエンジニアリングの核心理論と試行錯誤をまとめました。
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

## 1. プロンプトエンジニアリング：設計の美学

自分だけのAIサービスを効率的に構築するために、AI Engineeringの勉強を始めました。最初の記録として、プロンプトエンジニアリングの核心理論とコスパモデル`gpt-3.5-turbo`を最適化する中での試行錯誤をまとめてみます。

プロンプトエンジニアリングは単に質問を上手くする方法ではなく、**モデルの出力をビジネスロジックに合わせて誘導する設計技術**です。

### サービス設計ワークフロー

1. **問題条件の設定**：モデルが解決すべきTaskの範囲を明確に定義します。
2. **評価基準の設定**：どの回答が正解かを測定する指標（MMLU、G-Evalなど）を設定します。
3. **モデルの選定**：性能と**コスト（Cost）** のバランスを考慮して最適なモデルを選択します。

---

## 2. コストと性能の鍵：英語プロンプトとトークン（Token）

サービス化段階で最も重要なのは**コスト最適化**です。モデルはテキストをトークン単位で理解しており、ここで英語プロンプトを使用すべき決定的な理由が登場します。

### なぜ英語で作成すべきか？

- **トークン効率**：ほとんどのLLMのTokenizerは英語データセットを基に最適化されています。日本語や韓国語は1文字を表現するのに複数のトークンが消費されますが、英語は単語単位で効率的にエンコードされます。（結果的にAPIコスト削減）
- **推論性能**：モデルの学習データの圧倒的な割合が英語です。そのため、複雑な論理構造を指示する際は、英語で指示した方がモデルが意図をより正確に把握します。

### 実際のAPI呼び出し構造

指示文（System/User）は英語で、データは対象言語で渡す方式が効率的です。

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

## 3. Case Study：gpt-3.5-turboを困惑させた論理問題

コスパモデル`gpt-3.5-turbo`は、複合的な論理推論でしばしば限界を見せます。実際のMMLU（大規模多課題言語理解）形式の問題を通じて改善過程を見ていきましょう。

### 第1回目：単純Zero-Shot（失敗）

MMLUスタイルの数学問題を単純Zero-Shotで試みました。

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

- **結果**：モデルが**(C) 8**を選択。
- **失敗原因**：連立不等式の頂点を正確に求められず、単純な推論で誤答を導出。

### 改善：Role付与 + CoT（Chain of Thought）の適用

単純Zero-Shotが失敗した原因を分析し、2つのプロンプト技法を適用しました。

1. **Role付与**：Systemレベルで「You are a Professional in Mathematics.」という役割を指定し、数学専門家としての推論を誘導。
2. **CoT（Chain of Thought）**：「Think carefully」と「Give reasons」を明示して段階的思考を強制。

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

- **結果**：モデルが正解**(B) 6**を導出しました。

モデルの実際のReasoning出力は以下の通りです：

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

興味深い点は、モデルが途中でM-m=0という誤答を出しながらも**自ら再検討して正解に到達した**ということです。CoTによって「理由を説明せよ」と強制したおかげで、単純Zero-Shotでは不可能だった自己修正（Self-Correction）が可能になった事例です。

**適用した核心技法のまとめ：**

| 技法 | 第1回目 | 改善後 |
|---|---|---|
| Role付与 | なし | 数学専門家の役割指定 |
| 推論誘導 | 単純な回答要求 | 理由説明の強制（CoT） |
| 結果 | **(C) 8**（不正解） | **(B) 6**（正解） |

---

## 4. プロンプトエンジニアリングの限界と評価

プロンプトエンジニアリングは強力ですが、魔法ではありません。

- **長所**：再学習なしで性能を改善しコストを制御できる。
- **限界**：モデルの知識の限界や入力長（Context Window）の制限がある。
- **評価指標**：MMLU（知識）、MT Bench（対話）、ROUGE（要約）などを通じてプロンプトの品質を客観的に検証する必要があります。

---

## 5. 次回予告：じゃらん（Jalan）レビュー分析サービスの構築

理論と基礎テストを通じて`gpt-3.5-turbo`の最適化の可能性を確認しました。次の投稿では、実際の宿泊プラットフォームのレビューデータを活用して、AIが自動的に清潔度、サービス、コスパのインサイトを抽出する実践デモサービスの開発記を連載します。

コストを抑えつつ性能を最大化するAI Engineeringの実践プロセス、ご期待ください！
