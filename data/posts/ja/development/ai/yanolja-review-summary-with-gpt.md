---
title: "プロンプトエンジニアリング：理論から実践サービス構築まで（2）"
date: 2026-03-01 13:54
excerpt: Yanoljaの宿泊施設レビューをクローリングし、GPT-3.5-Turboで自動要約するパイプラインを構築。Few-shot PromptingとPairwise Evaluationで品質を改善した過程をまとめました。
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

# GPTを活用したYanolja宿泊施設レビュー自動要約サービス構築記

宿泊施設を予約する際、数百件のレビューを一つ一つ読むのはかなり面倒なことです。「この宿のレビューをまとめてほしい」というシンプルなアイデアから出発し、Yanoljaの宿泊施設レビューをクローリングしてGPTで自動要約するパイプラインを構築した過程をまとめます。

---

## パイプライン全体概要

```
[クローリング] → [前処理] → [プロンプトエンジニアリング] → [要約生成] → [評価] → [デモデプロイ]
```

1. **クローリング**：Selenium + BeautifulSoupでYanoljaレビューを収集
2. **前処理**：日付/長さフィルタリング、ポジティブ・ネガティブレビューの分離
3. **要約**：GPT-3.5-Turbo + Few-shot Prompting
4. **評価**：MT-Benchスタイル Pairwise Evaluation（GPT-4o）
5. **デモ**：Gradio Webインターフェース

---

## 1. データ収集 — Yanoljaレビュークローリング

Yanoljaのレビューページはスクロール時に動的にレビューを読み込むため、単純なHTTPリクエストだけではデータを取得できません。Seleniumでブラウザを制御してスクロールを繰り返した後、BeautifulSoupでHTMLを解析しました。

```python
def crawl_yanolja_reviews(name, url):
    review_list = []
    driver = webdriver.Chrome()
    driver.get(url)

    time.sleep(3)

    # スクロールを繰り返して動的に読み込まれたレビューをすべて取得
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

収集したデータの形式は以下の通りです。

```json
[
    {"review": "きれいでとても良かったです", "stars": 5, "date": "22時間前"},
    {"review": "立地も良く、宿もきれいで良いです〜", "stars": 5, "date": "2日前"},
    ...
]
```

星評価のカウントロジックは少しユニークです。Yanoljaでは星をSVGの`<path>`でレンダリングしているため、`fill="currentColor"`属性があり`fill-rule`属性がないpath要素の数を数えることで星の数を抽出しました。

仁寺洞、板橋、龍山の3拠点のレビューを収集し、実験に活用しました。

---

## 2. データ前処理

収集したレビューをそのままモデルに入力するとノイズが多く、要約品質が低下します。以下の基準で前処理を行いました。

```python
def preprocess_reviews(path='./res/reviews.json'):
    with open(path, 'r', encoding='utf-8') as f:
        review_list = json.load(f)

    reviews_good, reviews_bad = [], []

    current_date = datetime.datetime.now()
    date_boundary = current_date - datetime.timedelta(days=6*30)

    for r in review_list:
        review_date = parser.parse(r['date'])

        # 1) 直近6ヶ月以内のレビューのみ使用
        if review_date < date_boundary:
            continue
        # 2) 30文字未満の短いレビューを除去
        if len(r['review']) < 30:
            continue

        # 3) 星評価に基づいてポジティブ/ネガティブを分離
        if r['stars'] == 5:
            reviews_good.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')
        else:
            reviews_bad.append('[REVIEW_START]' + r['review'] + '[REVIEW_END]')

    # 4) 最大50件に制限
    reviews_good = reviews_good[:50]
    reviews_bad = reviews_bad[:50]

    return '\n'.join(reviews_good), '\n'.join(reviews_bad)
```

### 前処理基準

| 基準 | 理由 |
|------|------|
| 直近6ヶ月以内 | 古いレビューは現在の宿泊施設の状態を反映していない |
| 30文字以上 | 「良い」「きれい」のような短いレビューは要約の役に立たない |
| 5つ星 vs 4つ星以下 | ポジティブ/ネガティブの要約を別々に生成するため |
| 最大50件 | トークン制限内で処理するための制限 |

各レビューは`[REVIEW_START]`と`[REVIEW_END]`の区切り文字で囲み、モデルが個々のレビューの境界を明確に認識できるようにしました。

実際の仁寺洞データでは、全220件のレビュー中、5つ星レビュー177件、4つ星以下レビュー43件という分布が確認されました。

---

## 3. プロンプトエンジニアリング — 反復的改善プロセス

プロンプトエンジニアリングはこのプロジェクトの核心でした。ベースラインから始めて4段階にわたりプロンプトを改善し、各段階で評価を通じて効果を検証しました。

### 3-1. ベースライン — シンプルな指示

```python
PROMPT_BASELINE = "以下の宿泊施設レビューを5文以内で要約してください："
```

最もシンプルなプロンプトから始めました。GPT-3.5-Turboに特別な条件なしで要約を依頼したものです。

**結果**：10戦10勝（vs. Yanolja実際の要約）

意外にもシンプルなプロンプトでも既存のYanolja要約より良い結果を示しました。しかし、出力形式が一貫せず、敬語とタメ口が混在する問題がありました。

### 3-2. 改善1 — 出力条件の明示

```python
prompt = """あなたは要約の専門家です。ユーザーの宿泊施設レビューが与えられた時、
要約することがあなたの目標です。

要約結果は以下の条件を満たす必要があります：
1. すべての文は常に敬語で終わること。
2. 宿泊施設を紹介するトーン＆マナーで作成してください。
  2-1. 良い例
    a) 全体的に良い宿泊施設で、防音も良かったという評価です。
    b) 再訪予定だという評価が存在します。
  2-2. 悪い例
    a) 良い宿泊施設で、防音も良かったです。
    b) 再訪予定です。
3. 要約結果は最低2文、最大5文で作成してください。

以下の宿泊施設レビューを要約してください："""
```

役割付与、敬語、トーン＆マナー、文数制限など具体的な条件を追加しました。特に「紹介するトーン＆マナー」を良い例・悪い例で区別して説明したのがポイントです。レビューをそのまま転記するのではなく、第三者が宿泊施設を紹介する感じの文体を誘導しました。

**結果**：10戦10勝（vs. Yanolja実際の要約）

### 3-3. 改善2 — 入力データ品質の向上

プロンプトではなく入力データ側を改善しました。先に説明した前処理ロジック（30文字未満除去、6ヶ月以内、50件制限）をこの段階で適用しました。

**結果**：10戦7勝3敗（vs. Yanolja実際の要約）

興味深いことに、パフォーマンスがむしろ低下しました。短いながらもキーワードを含んでいたレビュー（「立地最高」「きれいです」）が除去されたことで、要約の多様性が減少したと分析しました。しかし長期的にはノイズ除去は必要なプロセスであるため、前処理は維持し、プロンプト側で補完する方向で進めました。

### 3-4. 改善3 — Few-Shot Prompting

パフォーマンス低下を挽回するためにFew-shot Promptingを導入しました。GPT-4-Turboで板橋拠点レビューの「模範要約」を生成し、それを例として一緒に提供する方式です。

```python
# GPT-4-Turboで板橋レビューの模範要約を生成
reviews_1shot, _ = preprocess_reviews('./res/ninetree_pangyo.json')
summary_1shot = summarize(
    reviews_1shot, prompt, temperature=0.0, model='gpt-4-turbo-2024-04-09'
).choices[0].message.content

# 1-Shotプロンプト構成
prompt_1shot = f"""あなたは要約の専門家です。ユーザーの宿泊施設レビューが
与えられた時、要約することがあなたの目標です。以下はレビューと要約の例です。

例のレビュー：
{reviews_1shot}
例の要約結果：
{summary_1shot}

以下の宿泊施設レビューを要約してください："""
```

核心的なアイデアは**GPT-4の出力をGPT-3.5のガイドとして使用する**ことです。高価なGPT-4は例の生成に一度だけ使用し、実際のサービスではコスト効率の良いGPT-3.5を使用します。

**1-Shot結果**：10戦9勝1敗（vs. Yanolja実際の要約）

龍山拠点のレビューで2-Shotまで拡張してみました。

**2-Shot結果**：10戦9勝1敗（vs. Yanolja実際の要約）

1-Shotと2-Shotのパフォーマンス差がなかったため、最終的に1-Shotを採用しました。トークンコストに対するパフォーマンス向上がなければ、より少ない例を使うのが合理的です。

### プロンプト改善過程のまとめ

| バージョン | 戦略 | 勝/敗/引 | 備考 |
|------------|------|----------|------|
| ベースライン | シンプルな指示 | 10/0/0 | 形式の一貫性不足 |
| v1 | 条件明示 | 10/0/0 | トーン＆マナー改善 |
| v2 | 前処理強化 | 7/3/0 | データ品質↑、多様性↓ |
| v3（1-Shot） | Few-Shot | 9/1/0 | GPT-4の例を活用 |
| v4（2-Shot） | Few-Shot | 9/1/0 | 1-Shotとの差なし |

---

## 4. 評価方法 — MT-BenchスタイルPairwise Evaluation

LLMが生成した要約の品質をどう評価するか？BLEU、ROUGEのような従来のメトリクスはテキストの意味的品質を適切に反映できません。このプロジェクトではMT-Bench論文で提案された**Pairwise Comparison**方式を採用しました。

### なぜスコアリングではなく比較方式なのか？

- **スコア方式の問題**：「この要約は何点か？」 → 基準が曖昧（3点の要約って何？）
- **比較方式の利点**：「AとBどちらが優れているか？」 → 相対的な判断がより明確で一貫している

### 評価の実装

GPT-4oを審判（Judge）として使用し、2つの要約を比較します。

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

### 評価基準

GPT-4oが判断する基準は以下の6つです。

- **Helpfulness（有用性）**：ユーザーにとって実質的に役立つか
- **Relevance（関連性）**：レビュー内容と関連があるか
- **Accuracy（正確性）**：事実に基づいた要約か
- **Depth（深さ）**：十分な深さを持っているか
- **Creativity（創造性）**：単純な羅列ではなく総合的な要約か
- **Level of Detail（詳細度）**：適切なレベルのディテールを含んでいるか

### 大規模評価 — temperatureの活用

同じプロンプトから多様な出力を得るために、`temperature=1.0`で10個の要約を生成し、それぞれをYanoljaの実際の要約と比較して勝率を測定しました。

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

## 5. 要約生成

最終的な要約生成は、GPT-3.5-Turboを`temperature=0.0`で呼び出し、一貫した結果を保証します。

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

ポジティブレビューとネガティブレビューをそれぞれ別々に要約し、ユーザーが宿泊施設の長所と短所を一目で把握できるようにしました。

---

## 6. Gradioデモ

Gradioを使ってシンプルなWebインターフェースを構築しました。

```python
def run_demo():
    demo = gr.Interface(
        fn=fn,
        inputs=[gr.Radio(['仁寺洞', '板橋', '龍山'], label='宿泊施設')],
        outputs=[
            gr.Textbox(label='高評価の要約'),
            gr.Textbox(label='低評価の要約')
        ]
    )
    demo.launch(share=True)
```

ユーザーが宿泊施設を選択すると、クローリングされたレビューを前処理し、GPTでポジティブ/ネガティブの要約をそれぞれ生成して表示します。`share=True`で外部共有リンクも生成できます。

---

## 学んだこと

**データ品質とプロンプトはトレードオフの関係になり得る。**
前処理を強化するとノイズは減るが情報量も減少します。この時、Few-shot Promptingが効果的な補完手段となりました。

**GPT-4の出力をGPT-3.5のガイドとして使う戦略はコスト効率が良い。**
GPT-4で模範解答を一度生成しておき、実際のサービスではGPT-3.5で運用すれば、品質とコストの両方を確保できます。

**LLM評価にはPairwise Comparisonが直感的。**
スコアベースの評価より「AとBのどちらが優れているか」という比較方式の方が一貫性があり、解釈しやすかったです。

**Few-shotは1つで十分な場合がある。**
1-Shotと2-Shotのパフォーマンス差がありませんでした。例の数をむやみに増やすより、良い例を1つ厳選することが重要です。

---

## 使用技術スタック

| 区分 | 技術 |
|------|------|
| クローリング | Selenium、BeautifulSoup |
| 要約モデル | GPT-3.5-Turbo（サービス）、GPT-4-Turbo（例の生成） |
| 評価モデル | GPT-4o |
| プロンプト | Few-shot Prompting（1-Shot） |
| デモ | Gradio |
| 言語 | Python |
