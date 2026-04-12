---
title: FastAPI(2) HTTP Response의 구조
date: 2026-04-13 6:10
excerpt: FastAPI에서의 HTTP Response 방법(Header・Body・Swagger)에 대해 알아봅시다.
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

FastAPI에서의 Response도 일반적인 구조인 「Header ・ Body」 구조를 사용하고 있습니다. 이번에는 FastAPI에서 어떤 것이 Default 값으로 되어 있고, 무엇을 커스터마이즈할 수 있는지 살펴보겠습니다.

## Header

Header는 상태 코드 ・ Header 항목 추가 ・ 출력 형식 등을 변경할 수 있습니다.

### 상태 코드

일반적으로 FastAPI에서는 성공하면 200, 에러는 4xx 코드를 반환하지만, 그 외에도 구현자가 의도한 상태 코드를 지정할 수 있습니다.

```python
@app.get("/happy")
def happ(status_code=201):
	return "I'm so happy !"
```

위와 같이 하면, 기본 200번이 아닌 201로 Response가 오는 것을 확인할 수 있습니다.

### Header 항목

기본적인 Header 항목 외에도, 구현자가 Header 항목을 추가하는 것도 가능합니다.

```python
from fastapi import Response

@app.get("/header/{name}/{value}")
def header(name: str, value: str, response: Response):
	response.header[name] = value
	return "normal body"
```

이렇게 하면, Header 안에 `name: value` 값을 확인할 수 있습니다.

### 출력 형식

FastAPI는 Default로 JSONResponse를 반환하도록 설정되어 있어, 별도의 지정이 없으면 항상 content-type이 application/json이 됩니다. 하지만 FastAPI에서는 경우에 따라 다른 것을 보내는 경우도 있습니다. 예를 들어 파일 같은 것이죠. 파일 전송에 대해서는 나중에 자세히 살펴보기로 하고, 이번에는 PlainTextResponse를 보내는 케이스를 살펴보겠습니다.

```python
from fastapi.responses import PlainTextResponse 

@app.get("/hello", response_class=PlainTextResponse) 
def hello_world(): 
	return "Hello, World! This is plain text."
```

그 외에도 HTMLResponse・RedirectResponse・FileResponse・StreamingResponse를 선택할 수 있습니다.

## Body

FastAPI에서 Body를 Response할 때에도 여러 가지 설정을 변경할 수 있습니다.

### 타입 변경

지금까지 본 것처럼, FastAPI에서는 Default로 JSONResponse를 하고 있어, 문자열을 자동으로 JSON으로 변환한 후 출력하는 것을 확인했습니다.
하지만 Python의 경우 datetime 등을 JSON으로 변환할 때 에러가 발생하기도 하는데, FastAPI 내부에서 먼저 jsonable_encoder()를 사용해 JSONable이라는 구조로 변경한 후, 그것을 JSON 문자열로 변환하도록 하고 있기 때문에, FastAPI에서는 안정적으로 JSON으로 변경하는 것이 가능합니다.

![Encoder](/images/posts/ja/development/fastapi/2_jsonable_encoder.png)

### Model 타입과 response_model

모던한 Backend 프레임워크에서는 일반적으로 Model Class를 작성하고, 그 Model Class를 Response하도록 만드는 케이스가 많습니다. FastAPI에서도 이에 대응하고 있습니다.
예를 들어, 의료용 데이터에서 비밀 정보를 DB에 자동 생성한다고 합시다. 그때 Model을 나누어 사용하는 것이 가능하며, response_model로 지정할 수 있습니다.

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
	# Tag를 DB에서 가져오는 것이 일반적이지만, 테스트용
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

재미있는 점은, 코드적으로 Tag를 return하고 있음에도 불구하고, response_model을 TagOut으로 지정하고 있기 때문에, FastAPI에서 이것을 자동으로 TagOut으로 변환하여 Response한다는 것입니다.

## 그 외

FastAPI에서는 별도의 설정 없이도 자동으로 Swagger를 추가해 줍니다.

![Swagger](/images/posts/ja/development/fastapi/2_swagger.png)
