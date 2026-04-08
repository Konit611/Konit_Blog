---
title: FastAPI(1) HTTP Request的机制
date: 2026-04-08 20:00
excerpt: 一起来学习FastAPI中的HTTP Request方法（Path・Query・Header・Body）吧。
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

我打算学习FastAPI。原因是最近流行的AI Engineering主要基于Python，所以我希望在用Python构建服务器时使用最现代的框架。

之后，我希望能将其与AI Engineering结合起来。

今天是第一篇，让我们来学习FastAPI中的HTTP Request方法。

## FastAPI HTTP Request的机制

FastAPI提供了对HTTP Protocol的Header・Path・Query・Body的支持。虽然这是一个通用的机制，但这意味着FastAPI作为框架已经为此做好了充分的准备。

首先，让我们来看一下最基本的FastAPI结构。

这是一段向hi这个URI发送请求后返回'Hello World'的代码。

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet():
	return 'Hello World'
	
if __name__ == "__main__":
	import uvicorn
	# 文件名为hello.py，FastAPI的变量名为app。
	uvicorn.run("hello:app", reload=True)
```

这是非常通用的HTTP Protocol。创建FastAPI()，查看该应用，让uvicorn运行起来。

如上所述，FastAPI将HTTP Protocol分为Header・Path・Query・Body来使用。首先，按照容易理解的顺序，从Path开始看起。

### 使用Path

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi/{name}")
def greet(name: str):
	return f'Hello {name}'
	
# 以下省略
```

这样的话，例如向`/hi/John`这个URI发送请求，就会返回`Hello John`这个字符串。

### 使用Query

Query的情况稍微有些复杂。如下所示，如果函数的参数名与查询参数名一致，就可以直接使用。不需要额外的设置。简单来看是很方便的，但对不了解的人来说可能会有些困惑。

```python
import fastapi from FastAPI

app = FastAPI()
@app.get("/hi")
def greet(name: str):
	return f'Hello {name}'
	
# 以下省略
```

这样的话，例如向`/hi?name=John`这个URI发送请求，就会返回`Hello John`这个字符串。

### 使用Header

Header中经常会放入密钥等一起使用。使用方法与Query几乎相同，但有一点需要注意：由于Python特有的命名方式，所有Header的变量名都会被转换为蛇形命名法（snake_case）。

```python
import fastapi from FastAPI, Header

app = FastAPI()
@app.get("/hi")
def greet(name: str = Header()):
	return f'Hello {user_agent}' # 想要使用User-Agent这个Header时
	
# 以下省略
```

这样就可以看到访问`/hi`时的User-Agent Header的值。

### 使用Body

Get Request的规则是只返回结果，有些框架为了维护数据的幂等性（Idempotent），会禁止在Get中使用Body，FastAPI也是如此。
因此，即使是用于验证的代码，也需要使用post。

```python
import fastapi from FastAPI, Body

app = FastAPI()
@app.post("/hi")
def greet(name: str = Body(embed=True)):
	return f'Hello {name}'
	
# 以下省略
```

### 其他建议

1. 虽然有像Body使用时那样的强制规定，但基本上推荐使用RESTful API。
2. Query用于可选参数・Body用于大型输入。
3. 如果正确编写Type Hint，可以使用Pydantic等库进行预先类型检查。
