---
title: "FastAPI(1) How HTTP Requests Work"
date: 2026-04-08 20:00
excerpt: Let's learn about HTTP Request methods in FastAPI — Path, Query, Header, and Body.
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

I decided to study FastAPI. The reason is that AI Engineering, which has been trending recently, is primarily Python-based, so I wanted to use the most modern framework when building servers with Python.

I plan to combine it with AI Engineering later on.

Today, let's study the first topic: how HTTP Requests work in FastAPI.

## How FastAPI HTTP Requests Work

FastAPI provides support for using HTTP Protocol's Header, Path, Query, and Body. While this is a standard mechanism, it means that FastAPI has properly prepared these as framework features.

First, let's look at the most basic FastAPI setup.

This is code that returns 'Hello World' when a request is sent to the URI `/hi`.

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet():
	return 'Hello World'
	
if __name__ == "__main__":
	import uvicorn
	# The file is named hello.py and the FastAPI variable is named app.
	uvicorn.run("hello:app", reload=True)
```

It's a very standard HTTP Protocol setup. You create a FastAPI() instance, reference the app, and uvicorn runs it.

As mentioned above, FastAPI allows you to use HTTP Protocol through Header, Path, Query, and Body. Let's start with Path, as it's the easiest to understand.

### Using Path

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi/{name}")
def greet(name: str):
	return f'Hello {name}'
	
# Below is omitted
```

With this setup, if you send a request to the URI `/hi/John`, it will return the string `Hello John`.

### Using Query

Query is slightly tricky. As shown below, if the function parameter name matches the query parameter, it can be used directly. No additional configuration is needed. It looks simple at first glance, but it can be confusing for newcomers.

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet(name: str):
	return f'Hello {name}'
	
# Below is omitted
```

With this setup, if you send a request to the URI `/hi?name=John`, it will return the string `Hello John`.

### Using Header

Headers are often used to include keys and other data. The usage is almost the same as Query, but there's one thing to keep in mind: due to Python's naming conventions, all Header variable names are converted to snake_case.

```python
import fastapi from FastAPI, Header

app = FastAPI()
@app.get("/hi")
def greet(name: str = Header()):
	return f'Hello {user_agent}' # When you want to use the User-Agent header
	
# Below is omitted
```

This allows you to see the User-Agent Header value when accessing `/hi`.

### Using Body

Since GET requests are meant to only return results, some frameworks prevent using Body with GET to maintain data idempotency — FastAPI is one of them.
Therefore, even for testing code, you need to use POST.

```python
import fastapi from FastAPI, Body

app = FastAPI()
@app.post("/hi")
def greet(name: str = Body(embed=True)):
	return f'Hello {name}'
	
# Below is omitted
```

### Other Recommendations

1. While some things are enforced like with Body, it's generally recommended to follow RESTful API conventions.
2. Use Query for optional parameters and Body for large inputs.
3. If you write proper Type Hints, you can use libraries like Pydantic to perform type validation in advance.
