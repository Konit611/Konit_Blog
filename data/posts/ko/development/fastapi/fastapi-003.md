---
title: FastAPI(3) Pydantic으로 데이터 모델 관리
date: 2026-04-15 6:10
excerpt: FastAPI에 내장된 Pydantic을 사용한 데이터 모델 선언・JSON 변환・유효성 검사에 대해 알아봅시다.
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

Python은 타입에 엄격하지 않은 언어이며, mypy라는 별도의 라이브러리를 설치해서 사용하지 않는 한 타입 힌트를 작성해도 마치 타입 힌트를 무시하는 것처럼 동작합니다.

Web 앱에서는 데이터를 의미 단위로 그룹화해서 관리하는 경우가 많지만, Python에서는 일반적인 class를 만드는 방법으로는 Web 앱을 만들 때 여러 가지로 불편합니다. 그래서 database 등의 라이브러리가 등장하고 있지만, 여기서는 FastAPI 내부에 들어가 있는 Pydantic(파이댄틱)에 대해 알아봅시다.

## Pydantic으로 그룹화된 데이터를 관리하는 방법

Pydantic에서는 그룹화된 데이터를 편리하게 관리할 수 있으며, BaseModel을 사용해서 선언합니다.

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

## Response될 때는 자동으로 JSON화

이전 장에서도 확인했듯이 FastAPI의 Starlette 엔진은 JSON화를 자동으로 수행하도록 되어 있어, FastAPI에 내장된 Pydantic과의 궁합이 좋습니다.

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

## 유효성 검사

데이터의 타입은 물론, 값에 대한 유효성 검사도 이루어집니다.

### 타입이 맞지 않는 경우

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

### 값이 맞지 않는 경우

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
