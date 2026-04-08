---
title: FastAPI(1) HTTP Request의 구조
date: 2026-04-08 20:00
excerpt: FastAPI에서의 HTTP Request 방법(Path・Query・Header・Body)에 대해 알아봅시다.
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

FastAPI를 공부하려고 합니다. 그 이유는 최근 유행하고 있는 AI Engineering이 주로 Python 기반이기 때문에, Python으로 서버를 만들 때 가장 모던한 프레임워크를 사용하고 싶었기 때문입니다.

추후에 AI Engineering과 결합할 수 있도록 하고 싶습니다.

오늘은 그 첫 번째, FastAPI에서의 HTTP Request 방법에 대해 공부해봅시다.

## FastAPI HTTP Request의 구조

FastAPI에서는 HTTP Protocol의 Header・Path・Query・Body를 사용할 수 있도록 준비되어 있습니다. 이는 일반적인 구조이지만, 프레임워크로서 제대로 사용할 수 있도록 준비해두었다는 의미입니다.

먼저, 가장 기본적인 FastAPI의 구조를 살펴봅시다.

hi라는 URI에 요청을 보내면 'Hello World'가 반환되는 코드입니다.

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet():
	return 'Hello World'
	
if __name__ == "__main__":
	import uvicorn
	# 파일명이 hello.py이고 FastAPI의 변수명을 app으로 하고 있습니다.
	uvicorn.run("hello:app", reload=True)
```

아주 일반적인 HTTP Protocol이죠. FastAPI()를 만들고, 그 앱을 보고, uvicorn이 실행되도록 합니다.

위에서도 말씀드렸지만, FastAPI는 HTTP Protocol을 Header・Path・Query・Body로 나누어 사용할 수 있도록 하고 있습니다. 먼저, 이해하기 쉬운 순서로 Path부터 살펴봅시다.

### Path 사용하기

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi/{name}")
def greet(name: str):
	return f'Hello {name}'
	
# 아래는 생략
```

이렇게 하면, 예를 들어 `/hi/John`이라는 URI에 요청을 보내면 `Hello John`이라는 문자열을 반환합니다.

### Query 사용하기

Query의 경우 약간 복잡한데, 아래와 같이 함수의 인수 이름과 쿼리가 일치하면 그대로 사용할 수 있게 됩니다. 별도의 설정은 필요 없습니다. 간단하게 보면 편하지만, 모르는 사람에게는 좀 복잡하죠.

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet(name: str):
	return f'Hello {name}'
	
# 아래는 생략
```

이렇게 하면, 예를 들어 `/hi?name=John`이라는 URI에 요청을 보내면 `Hello John`이라는 문자열을 반환합니다.

### Header 사용하기

Header에는 키 등을 넣어 함께 사용하는 경우도 많죠. 사용 방법은 Query와 거의 동일하지만, 한 가지 알아두어야 할 점이 있습니다. Python 특유의 작성 방식으로 인해 모든 Header의 변수명을 스네이크 케이스로 변환한다는 것입니다.

```python
import fastapi from FastAPI, Header

app = FastAPI()
@app.get("/hi")
def greet(name: str = Header()):
	return f'Hello {user_agent}' # User-Agent라는 Header를 사용하고 싶을 때
	
# 아래는 생략
```

이렇게 하면, `/hi`에 접근했을 때의 User-Agent Header 값을 볼 수 있습니다.

### Body 사용하기

Get Request는 결과만 반환하는 것이 규칙이므로, 프레임워크에 따라서는 데이터의 Idempotent를 유지하기 위해 Get에서는 Body를 사용할 수 없도록 하는 프레임워크가 가끔 있는데, FastAPI도 그렇습니다.
따라서, 확인하기 위한 코드에서도 post를 사용해야 합니다.

```python
import fastapi from FastAPI, Body

app = FastAPI()
@app.post("/hi")
def greet(name: str = Body(embed=True)):
	return f'Hello {name}'
	
# 아래는 생략
```

### 기타 추천사항

1. Body를 사용할 때처럼 강제적인 것도 있지만, 기본적으로 RESTful API를 사용하는 것이 권장됩니다.
2. Query는 선택적 인수・Body는 큰 Input에 대해 사용합시다.
3. Type Hint를 제대로 작성하면, Pydantic 등의 라이브러리를 사용하여 사전에 타입 검사를 할 수 있습니다.
