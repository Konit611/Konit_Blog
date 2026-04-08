---
title: FastAPI(1) HTTP Requestの仕組み
date: 2026-04-08 20:00
excerpt: FastAPIでのHTTP Request方法（Path・Query・Header・Body）について学びましょう。
coverImage:
categories:
  - fastapi
tags:
  - FastAPI
  - Python
  - Web Framework
  - HTTP
author: Geunil Park
featured: false
---

FastAPIの勉強をしようと思いました。その理由は最近流行っているAI Engineeringが主にPythonであるため、Pythonでサーバーを作る際に一番モダンなフレームワークを使いたいと思ったからです。

後ほど、AI Engineeringと組み合わせるようにしたいと思います。

今日はその第一弾FastAPIでのHTTP Request方法に関して勉強しましょう

## FastAPI HTTP Requestの仕組み

FastAPIではHTTP ProtocolのHeader ・ Path ・ Query ・ Body を使えるように用意されております。これは一般的な仕組みではありますが、ちゃんと使えるようにフレームワークとして用意していると言う意味です。

まず、一番基本的なFastAPIの仕組みを見てみましょう。

hiというURIにリクエストを送ると'Hello World'が変えてくるようになるコードです。

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet():
	return 'Hello World'
	
if __name__ == "__main__":
	import uvicorn
	# ファイル名がhello.pyでFastAPIの変数名をappにしている。
	uvicorn.run("hello:app", reload=True)が
```

すごく一般的なHTTP Protocolですね。FastAPI()を作り、そのアプリを見て、uvicorn が走れるようになります。

上記でも言いましたが、FastAPIはHTTP ProtocolをHeader ・ Path ・ Query ・ Bodyに分けて使えるようにしております。まず、理解しやすい順として、Pathから見てみましょう。

### Pathを使う

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi/{name}")
def greet(name: str):
	return f'Hello {name}'
	
# 下は省略
```

こうしますと、例えば`/hi/John`というURIにリクエストを送りますと、`Hello John`という文字列を返します。

### Qeuryを使う

Queryの場合は少しややこしいのですが、下のように、関数の引数の名前とクエリが一致するとそれをそのまま使えるようになります。別途の設定は入りません。簡単に見ると楽ですが、知らない人にしてはややこしいですね。

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet(name: str):
	return f'Hello {name}'
	
# 下は省略
```

こうしますと、例えば`/hi?name=John`というURIにリクエストを送りますと、`Hello John`という文字列を返します。

### Headerを使う

Headerにはキーなどを入れて一緒に使う場合も多いですよね。使え方としてはQueryとほぼ一緒ですが、一つ押さえておく必要があります。それはPythonでは特有の書き方により、すべてのHeaderの変数名を全部スネークケースにしてしまうことです。

```python
import fastapi from FastAPI, Header

app = FastAPI()
@app.get("/hi")
def greet(name: str = Header()):
	return f'Hello {user_agent}' #User-AgentというHeaderを使いたいときに
	
# 下は省略
```

こうしますと、`/hi`に接近した時のUser-Agent Headerのバリューを見ることができます。

### Bodyを使う

Get Requestは結果だけを返すのが決まり事でして、フレームワークによってはデータのIdempotentを維持するために、GetではBodyを使えないようにするフレームワークがたまにありますが、FastAPIもそうです。
ですので、確認するためのコードでもpostを使う必要があります。

```python
import fastapi from FastAPI, Body

app = FastAPI()
@app.post("/hi")
def greet(name: str = Body(embed=True):
	return f'Hello {name}'
	
# 下は省略
```

### その他おすすめ

1. Bodyを使う時みたいに強制的になっているものもありますが、基本的にRESTful APIを使うことがおすすめされております。
2. Queryは選択的な引数・Bodyは大きいInputに対して使いましょう
3. Type Hintをちゃんと書いたら、Pydanticなどのライブラリーを使って事前にタイプ検査をすることができます。