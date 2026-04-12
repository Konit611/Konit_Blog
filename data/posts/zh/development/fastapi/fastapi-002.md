---
title: FastAPI(2) HTTP Response的机制
date: 2026-04-13 6:10
excerpt: 一起来学习FastAPI中的HTTP Response方法（Header・Body・Swagger）吧。
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

FastAPI中的Response也使用了常见的「Header ・ Body」结构。这次我们来看看FastAPI中哪些是默认值，以及可以自定义哪些内容。

## Header

Header可以修改状态码、添加Header项目、更改输出格式等。

### 状态码

一般来说，FastAPI在成功时返回200，错误时返回4xx代码，但除此之外，开发者也可以指定自己想要的状态码。

```python
@app.get("/happy")
def happ(status_code=201):
	return "I'm so happy !"
```

像上面这样设置后，可以确认Response返回的是201而不是默认的200。

### Header项目

除了基本的Header项目外，开发者也可以添加自定义的Header项目。

```python
from fastapi import Response

@app.get("/header/{name}/{value}")
def header(name: str, value: str, response: Response):
	response.header[name] = value
	return "normal body"
```

这样就可以在Header中看到 `name: value` 的值。

### 输出格式

FastAPI默认设置为返回JSONResponse，如果没有特别指定，content-type始终是application/json。但在FastAPI中，有时也需要发送不同的内容，比如文件。关于文件传输我们之后会详细介绍，这次先来看看发送PlainTextResponse的情况。

```python
from fastapi.responses import PlainTextResponse 

@app.get("/hello", response_class=PlainTextResponse) 
def hello_world(): 
	return "Hello, World! This is plain text."
```

此外还可以选择HTMLResponse・RedirectResponse・FileResponse・StreamingResponse。

## Body

在FastAPI中Response Body时，也可以更改各种设置。

### 类型转换

如我们所见，FastAPI默认使用JSONResponse，会自动将字符串转换为JSON后输出。
但在Python中，将datetime等类型转换为JSON时可能会出错。FastAPI内部首先使用jsonable_encoder()将数据转换为JSONable格式，然后再将其转换为JSON字符串，因此FastAPI可以稳定地进行JSON转换。

![Encoder](/images/posts/ja/development/fastapi/2_jsonable_encoder.png)

### Model类型与response_model

在现代Backend框架中，通常会创建Model Class，并将该Model Class作为Response返回。FastAPI也支持这种方式。
例如，假设要在医疗数据中自动生成一些秘密信息到DB中。这时可以将Model分开使用，并指定为response_model。

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
	# 一般来说会从DB获取Tag，但这里是测试用的
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

有趣的是，虽然代码中return的是Tag，但由于response_model指定为TagOut，FastAPI会自动将其转换为TagOut后再Response。

## 其他

FastAPI无需任何额外设置，就会自动添加Swagger。

![Swagger](/images/posts/ja/development/fastapi/2_swagger.png)
