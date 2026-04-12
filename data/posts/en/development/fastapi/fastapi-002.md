---
title: "FastAPI(2) How HTTP Responses Work"
date: 2026-04-13 6:10
excerpt: Let's learn about HTTP Response methods in FastAPI — Header, Body, and Swagger.
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

Responses in FastAPI also use the common "Header / Body" structure. This time, let's look at what the default values are in FastAPI and what can be customized.

## Header

You can modify the status code, add Header fields, and change the output format.

### Status Code

By default, FastAPI returns 200 for success and 4xx codes for errors, but you can also specify custom status codes as intended by the developer.

```python
@app.get("/happy")
def happ(status_code=201):
	return "I'm so happy !"
```

With the above code, you can confirm that the response comes back with 201 instead of the default 200.

### Header Fields

In addition to the default Header fields, developers can also add custom Header fields.

```python
from fastapi import Response

@app.get("/header/{name}/{value}")
def header(name: str, value: str, response: Response):
	response.header[name] = value
	return "normal body"
```

This way, you can see the `name: value` pair in the Header.

### Output Format

FastAPI is configured to return JSONResponse by default, so the content-type is always application/json unless otherwise specified. However, there are cases where FastAPI needs to send something different — for example, files. We'll look at file transfer in detail later, but for now let's look at the case of sending a PlainTextResponse.

```python
from fastapi.responses import PlainTextResponse 

@app.get("/hello", response_class=PlainTextResponse) 
def hello_world(): 
	return "Hello, World! This is plain text."
```

Other options include HTMLResponse, RedirectResponse, FileResponse, and StreamingResponse.

## Body

There are various settings you can change when sending a Body in FastAPI responses.

### Type Conversion

As we've seen, FastAPI uses JSONResponse by default and automatically converts strings to JSON before outputting them.
However, in Python, converting types like datetime to JSON can cause errors. FastAPI internally uses jsonable_encoder() to first convert data into a JSONable format, then converts it to a JSON string. This allows FastAPI to reliably convert data to JSON.

![Encoder](/images/posts/ja/development/fastapi/2_jsonable_encoder.png)

### Model Types and response_model

In modern backend frameworks, it's common to create Model Classes and have them returned as Responses. FastAPI supports this as well.
For example, let's say you auto-generate some secret information in a DB for medical data. In this case, you can separate the Models and specify a response_model.

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
	# Normally you'd fetch the Tag from DB, but this is for testing
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

The interesting thing is that even though the code returns a Tag, because response_model is set to TagOut, FastAPI automatically converts it to TagOut before sending the Response.

## Others

FastAPI automatically adds Swagger without any additional configuration.

![Swagger](/images/posts/ja/development/fastapi/2_swagger.png)
