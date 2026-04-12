---
title: FastAPI(2) HTTP Responseの仕組み
date: 2026-04-13 6:10
excerpt: FastAPIでのHTTP Response方法（Header・Body・Swagger）について学びましょう。
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

FastAPIでのResponseも一般的な仕組みである「Header ・ Body」構造を使っております。この度はFastAPIで何かDefaultの値になっていて、何をカスタマイズできるのかを見ていきましょう。

## Header

Headerは状態コード ・ Header項目の追加 ・ 出力形式などを変更することができます。

### 状態コード

一般的にFastAPIでは成功したら200で、エラーは4xxコードを出しますが、それ以外にも実装者が意図した状態コードを指定することができます。

```python
@app.get("/happy")
def happ(status_code=201):
	return "I'm so happy !"
```

上のようにすると、基本の200番ではなく、201で返答がResponseが来ることを確認できます。

### Header項目

基本的なHeader項目以外にも、実装者がHeader項目を追加することも可能です。

```python
from fastapi import Response

@app.get("/header/{name}/{value}")
def header(name: str, value: str, response: Response):
	response.header[name] = value
	return "normal body"
```

こうしますと、Headerの中身に `name: value`の値を確認することができます。

### 出力形式

FastAPIはDefaultでJSONResponseを返すような設定になっており、別途の指定がない場合いつもcontent-typeがapplication/jsonになります。しかし、FastAPIでは場合によって違うものを送る場合もあります。例えばファイルとかですね。ファイルの移行に対しては後ほど詳しく見ていくことにして、今回はPlainTextResponseをお送りするケースを見ておきましょう。

```python
from fastapi.responses import PlainTextResponse 

@app.get("/hello", response_class=PlainTextResponse) 
def hello_world(): 
	return "Hello, World! This is plain text."
```

他にも、HTMLResponse・RedirectResponse・FileResponse・StreamingResponseが選択できます。

## Body

FastAPIでBodyをResponseする際にも色々設定を変更することができます。

### タイプ変更

今まで見たように、FastAPIではDefaultでJSONResponseをしており、文字列を自動でJSONに変換してから出すことを確認できました。
しかし、Pythonの場合datetimeなどをJSONに変換する際にエラーがなったりしますが、FastAPIの内部でまず、jsonable_encoder()を使い、JSONableという仕組みに変更した後、それをJSON文字列に変換するようにしているため、FastAPIでは安定的にJSONに変更することが可能になります。

![Encoder](/images/posts/ja/development/fastapi/2_jsonable_encoder.png)

### Model タイプとresponse_model

ModernなBackendフレームワークでは一般的にModel Classを作成し、そのModel ClassをResponseするように作るケースが多いのです。FastAPIにもそれに対応しております。
例えば、医療用のデータで何か秘密情報をDBに自動生成するとしましょう。その際にModelを分けて使うことが可能で、response_modelとして指定することができます。

```python
# model/tag.py

from datetime import datetime
from pydantic import BaseModel

class TagIn(BaseModel):
	tag: str
	
class Tag(BaseModel):
	tag: str
	created: datetime
	secret: str
	
class TagOut(BaseModel):
	tag: str
	created: datetime
```

```python
# service/tag.py

from datetime import datetime
from model.tag import Tag
	
def get(tag_str: str) -> Tag:
	# TagをDBから取ってくるのが一般的だが、テストのためのもの
	return Tag(tag=tag_str, created=datetime.utcnow(), secret="")
```

```python
from datetime import datetime
from model.tag import Tag, TagOut
import service.tag as service
from fastapi import FastAPI

app = FastAPI()

@app.get('/{tag_str}', response_model=TagOut)
def get_one(tag_str: str) -> TagOut:
	tag: Tag = service.get(tag_str)
	return tag
```

面白いのは、コード的にTagをreturnしているのも関わらず、response_modelをTagOutに指定しているため、FastAPIでこれを自動でTagOutに変換してResponseすることです。

## そのほか

FastAPIでは別途の設定がなくても、自動でSwaggerを追加してくれます。

![Swagger](/images/posts/ja/development/fastapi/2_swagger.png)