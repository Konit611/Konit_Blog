---
title: FastAPI(3) 使用Pydantic管理数据模型
date: 2026-04-15 6:10
excerpt: 一起来学习如何使用FastAPI内置的Pydantic声明数据模型、进行JSON转换以及有效性校验吧。
coverImage:
categories:
  - fastapi
tags:
  - FastAPI
  - Python
  - Pydantic
  - Web Framework
author: Geunil Park
featured: false
---

Python是一种对类型不严格的语言，除非安装并使用mypy这样的额外库，否则即使写了类型提示，运行时也几乎像是忽略类型提示一样。

在Web应用中，经常需要将数据按语义单位分组管理，但在Python中，使用普通的class来实现这一点在构建Web应用时会有诸多不便。为此出现了database等库，这里我们来了解一下FastAPI内部所集成的Pydantic（派丹蒂克）。

## 使用Pydantic管理分组数据的方法

Pydantic可以方便地管理分组数据，使用BaseModel来声明。

```python
# model.py
from pydantic import BaseModel

class Creature(BaseModel):
	name: str
	country: str
	area: str
	description: str
	aka: str
```

## Response时自动JSON化

正如前一章所确认的，FastAPI的Starlette引擎会自动进行JSON转换，与FastAPI内置的Pydantic配合得很好。

```python
# data.py

from model import Creature

creatures: list[Creature] = [
	Creature(
		name="yeti",
		country="CN",
		area="Himalayas",
		description="Hirsute Himalayan",
		aka="Abominable Snowman"
	),
	Creature(
		name="sasquatch",
		country="US",
		area="*",
		description="Yeti's Cousin Eddie",
		aka="Bigfoot"
	)
]
```

```python
# web.py
from model import Creature
from fastapi import FastAPI

app = FastAPI()

@app.get("/creature")
def get_all() -> list[Creature]:
	from data import creatures
	return creatures
```

```shell
# result

$ http http://localhost:8000/creature

HTTP/1.1 200 OK
content-length: 211
content-type: application/json
date: Tue, 14 Apr 2026 21:51:21 GMT
server: uvicorn

[
    {
        "aka": "Abominable Snowman",
        "area": "Himalayas",
        "country": "CN",
        "description": "Hirsute Himalayan",
        "name": "yeti"
    },
    {
        "aka": "Bigfoot",
        "area": "*",
        "country": "US",
        "description": "Yeti's Cousin Eddie",
        "name": "sasquatch"
    }
]

```

## 有效性校验

不仅会校验数据的类型，还会对值进行有效性校验。

### 类型不匹配的情况

```python
# test1.py

from model import Creature

dragon = Creature(
	name="dragon",
	description=['Fire-breathing, winged, scaly beast'],
	country="*",
	area="*",
	aka="Drake"
)
```

```shell
$ python3 test1.py

Traceback (most recent call last):
  File "/Users/geunil/Study/FastAPI/ch5/test1.py", line 3, in <module>
    dragon = Creature(
        name="dragon",
    ...<3 lines>...
        aka="Drake"
    )
  File "/Users/geunil/Study/FastAPI/ch3/.venv/lib/python3.13/site-packages/pydantic/main.py", line 250, in __init__
    validated_self = self.__pydantic_validator__.validate_python(data, self_instance=self)
pydantic_core._pydantic_core.ValidationError: 1 validation error for Creature
description
  Input should be a valid string [type=string_type, input_value=['Fire-breathing, winged, scaly beast'], input_type=list]
    For further information visit https://errors.pydantic.dev/2.12/v/string_type

```

### 值不匹配的情况

```python
# new_model.py

from pydantic import BaseModel, constr

class NewCreature(BaseModel):
	name: str
	country: constr(min_length=2)
	area: str
	description: str
	aka: str
```

```python
# test2.py
from new_model import NewCreature

dragon = NewCreature(
	name="dragon",
	description="Fire-breathing, winged, scaly beast",
	country="*",
	area="Worldwide",
	aka="Drake"
)
```

```shell
$ python3 test2.py
Traceback (most recent call last):
  File "/Users/geunil/Study/FastAPI/ch5/test2.py", line 3, in <module>
    dragon = NewCreature(
        name="dragon",
    ...<3 lines>...
        aka="Drake"
    )
  File "/Users/geunil/Study/FastAPI/ch3/.venv/lib/python3.13/site-packages/pydantic/main.py", line 250, in __init__
    validated_self = self.__pydantic_validator__.validate_python(data, self_instance=self)
pydantic_core._pydantic_core.ValidationError: 1 validation error for NewCreature
country
  String should have at least 2 characters [type=string_too_short, input_value='*', input_type=str]
    For further information visit https://errors.pydantic.dev/2.12/v/string_too_short
```
