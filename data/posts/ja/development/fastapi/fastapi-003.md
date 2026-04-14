---
title: FastAPI(3) Pydanticでのデータモデル管理
date: 2026-04-15 6:10
excerpt: FastAPIに内在されているPydanticを使ったデータモデルの宣言・JSON変換・有効性検査について学びましょう。
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

Pythonはタイプに厳しくない言語であり、mypyという別途のライブラリを設置し使わない限りタイプヒントを書いてもまるでタイプヒントを無視するように動く

Webアプリではデータを意味単位でグループ化して管理する場合が多いが、Pythonでは一般的なclassを作る方法では、Webアプリを作る際に色々と不便がある。そこでdatabaseなどライブラリが登場してるが、ここではFastAPIの内部に入ったPydantic(パイダンティック)に対して調べてみよう

## Pydanticでグループ化されたデータを管理する方法

Pydanticではグループ化されたデータを便利に管理することが可能であり、BaseModelを使って宣言する。

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

## Responseされる際には自動でJSON化

前章でも確認したようにFastAPIのStarletteエンジンはJSON化を自動で行うことになっており、FastAPIに内在されているPydanticとの相性がいい。

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

## 有効性検査

データのタイプは勿論、値に対する有効性の検査も行われる。

### タイプが合わないケース

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

### 値が合わないケース

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