---
title: "FastAPI(3) Managing Data Models with Pydantic"
date: 2026-04-15 6:10
excerpt: Let's learn how to declare data models, convert them to JSON, and validate them using Pydantic, which is built into FastAPI.
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

Python is not a strictly typed language, and unless you install and use a separate library like mypy, type hints behave as if they're being ignored.

In web apps, data is often grouped and managed in semantic units, but in Python, using ordinary classes to do this is inconvenient when building web apps. Libraries like database have emerged to address this, but here let's take a look at Pydantic, which is built into FastAPI.

## How to manage grouped data with Pydantic

Pydantic lets you conveniently manage grouped data, and you declare it using BaseModel.

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

## Automatic JSON conversion on Response

As we confirmed in the previous chapter, FastAPI's Starlette engine performs JSON conversion automatically, which makes it work well with Pydantic, which is built into FastAPI.

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

## Validation

Not only the data types but also the values themselves are validated.

### When the type doesn't match

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

### When the value doesn't match

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
