# Module 1: FastAPI Fundamentals

## The Analogy: The Restaurant Dispatcher

Imagine a busy restaurant with one order dispatcher:

- **Old way (Flask/Django sync)**: Dispatcher takes order, goes to kitchen, waits for food, comes back. Can handle one table at a time. Inefficient.
- **New way (FastAPI async)**: Dispatcher takes order, sends it to kitchen, immediately takes the next order. Kitchen sends back each completed dish as it's ready. Can handle 10 tables simultaneously.

FastAPI is the efficient dispatcher. Instead of blocking on one request, it handles many concurrently.

## What FastAPI Is

**FastAPI** is:

1. **A web framework** — Receives HTTP requests, routes them to handlers, returns responses
2. **Built on ASGI** — Asynchronous Server Gateway Interface (async by default)
3. **Typed** — Uses Python type hints for validation and documentation
4. **Fast** — Comparable to Go and Node.js in performance benchmarks
5. **Auto-documented** — Generates interactive Swagger UI from your code

### The Three Layers

```
┌─────────────────────────────────┐
│   Starlette (HTTP framework)   │
│   (routing, request/response)   │
├─────────────────────────────────┤
│  Pydantic (data validation)    │
│  (validates input, serializes)  │
├─────────────────────────────────┤
│      Your routes and logic      │
└─────────────────────────────────┘
```

FastAPI sits on top of Starlette and adds Pydantic. You write the handler logic and let them handle the rest.

## ASGI vs WSGI: Why It Matters

### WSGI (Old Standard)

```
Request → Middleware → Handler (blocking!) → Response
```

While one request is being processed (waiting for database), the whole worker process is stuck. You need multiple processes (10 Gunicorn workers).

### ASGI (New Standard)

```
Request → Middleware → Handler (async) → Response
         ↓
     Request 2 → Middleware → Handler (async) → Response
```

One worker handles multiple requests concurrently. While Request 1 waits for the database, Request 2 is already being processed.

**Result:**
- **Flask (WSGI)**: 4 Gunicorn workers needed to handle 4 concurrent requests
- **FastAPI (ASGI)**: 1 Uvicorn worker handles 100+ concurrent requests

This is why FastAPI scales better.

## Your First FastAPI App

### Installation

```bash
pip install fastapi uvicorn[standard]
```

### main.py

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/health")
async def health():
    return JSONResponse({"status": "ok"})

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Running It

```bash
uvicorn main:app --reload
```

The `--reload` flag restarts the server when you change code (development only).

Open http://localhost:8000 in your browser. You'll see:

```json
{"message": "Hello World"}
```

### The /docs Endpoint

Go to http://localhost:8000/docs

You'll see **Swagger UI**—an interactive API documentation page. You can:
- See all endpoints
- Read descriptions
- Click "Try it out" and test endpoints
- See request/response examples

This is auto-generated from your code. No manual documentation needed.

## The Request/Response Cycle

When you curl http://localhost:8000/hello/alice:

```
1. Request arrives at Nginx
   GET /hello/alice HTTP/1.1
   Host: localhost:8000

2. Nginx forwards to Uvicorn (reverse proxy)

3. Uvicorn router matches to @app.get("/hello/{name}")

4. FastAPI validates path parameter (extracts "alice" as name)

5. Route handler executes:
   async def hello(name: str):
       return {"greeting": f"Hello {name}"}

6. FastAPI serializes response to JSON:
   {"greeting": "Hello alice"}

7. Uvicorn sends back to Nginx:
   HTTP/1.1 200 OK
   Content-Type: application/json
   {"greeting": "Hello alice"}

8. Browser receives and displays
```

The key step is **FastAPI validates path parameters automatically** using type hints. If you pass `/hello/123`, it validates that 123 can be converted to whatever type you specified.

## Route Decorators: GET, POST, PUT, DELETE, PATCH

### GET — Retrieve Data

```python
@app.get("/items/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id}
```

### POST — Create Data

```python
@app.post("/items")
async def create_item(name: str, price: float):
    return {"name": name, "price": price, "created": True}
```

### PUT — Replace Data

```python
@app.put("/items/{item_id}")
async def update_item(item_id: int, name: str):
    return {"item_id": item_id, "name": name}
```

### DELETE — Remove Data

```python
@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    return {"item_id": item_id, "deleted": True}
```

### PATCH — Partial Update

```python
@app.patch("/items/{item_id}")
async def patch_item(item_id: int, name: str = None):
    if name:
        return {"item_id": item_id, "name": name}
    return {"item_id": item_id}
```

## Path Parameters vs Query Parameters vs Request Body

### Path Parameters

Part of the URL itself:

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # user_id comes from the URL path
    return {"user_id": user_id}
```

Usage: `GET /users/42`

### Query Parameters

After the `?` in the URL:

```python
@app.get("/search")
async def search(q: str, limit: int = 10):
    # q is required, limit defaults to 10
    return {"query": q, "limit": limit}
```

Usage: `GET /search?q=python&limit=5`

### Request Body

Sent in the HTTP body (usually JSON):

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    tax: float = 0.0

@app.post("/items")
async def create_item(item: Item):
    return item
```

Usage:
```bash
curl -X POST http://localhost:8000/items \
  -H "Content-Type: application/json" \
  -d '{"name":"Book","price":15.99}'
```

## Status Codes

HTTP status codes tell the client if the request succeeded:

| Code | Meaning | Use |
|------|---------|-----|
| 200 | OK | Successful GET/PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input (user's fault) |
| 401 | Unauthorized | Missing/invalid auth |
| 403 | Forbidden | Authenticated but not permitted |
| 404 | Not Found | Resource doesn't exist |
| 422 | Unprocessable Entity | Validation error (Pydantic) |
| 500 | Server Error | Bug in your code |
| 503 | Service Unavailable | Database down, etc. |

Set a status code in FastAPI:

```python
@app.post("/items", status_code=201)
async def create_item(name: str):
    return {"name": name, "created": True}
```

## Swagger UI: Auto-Generated Docs

FastAPI reads your Python code and generates documentation automatically.

```python
@app.get("/users/{user_id}", summary="Get a user", tags=["users"])
async def get_user(user_id: int):
    """
    Get a user by their ID.
    
    - **user_id**: The unique identifier of the user
    
    Returns the user object with id, name, email.
    """
    return {"user_id": user_id, "name": "Alice"}
```

The docstring becomes the endpoint description in Swagger. Tags organize endpoints.

Visit http://localhost:8000/docs and you'll see all this information.

!!! tip
    The Swagger UI is incredibly useful for debugging. You can test every endpoint without writing curl commands.

## Hands-On Lab

### Lab 1.1: Create a Simple FastAPI App

```bash
mkdir -p fastapi_lab
cd fastapi_lab
```

#### Step 1: Create main.py

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI(
    title="My API",
    description="A simple API for learning FastAPI",
    version="1.0.0"
)

@app.get("/")
async def root():
    """The root endpoint."""
    return {"message": "Welcome to my API"}

@app.get("/health")
async def health():
    """Health check endpoint."""
    return JSONResponse({"status": "ok"})

@app.get("/hello/{name}")
async def greet(name: str):
    """Greet someone by name."""
    return {"greeting": f"Hello, {name}!"}

@app.post("/echo")
async def echo(message: str):
    """Echo back the message you send."""
    return {"echo": message}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### Step 2: Install and Run

```bash
pip install fastapi uvicorn[standard]
python main.py
```

Visit http://localhost:8000/docs

#### Step 3: Test with curl

```bash
# Test root
curl http://localhost:8000/

# Test health
curl http://localhost:8000/health

# Test hello with a path parameter
curl http://localhost:8000/hello/Alice

# Test echo with a query parameter
curl -X POST "http://localhost:8000/echo?message=Hello"
```

#### Step 4: Explore Swagger UI

Go to http://localhost:8000/docs

- Click each endpoint to see its documentation
- Click "Try it out" to test endpoints from the browser
- Change path/query parameters and see the curl command at the bottom

### Lab 1.2: Add Query Parameters with Defaults

Add this to main.py:

```python
@app.get("/search")
async def search(q: str, skip: int = 0, limit: int = 10):
    """Search with optional pagination."""
    return {
        "query": q,
        "skip": skip,
        "limit": limit,
        "results": []  # Placeholder
    }
```

Test:

```bash
curl "http://localhost:8000/search?q=python"
curl "http://localhost:8000/search?q=python&skip=5&limit=20"
```

### Lab 1.3: Request Body with Pydantic

Add to main.py:

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    tax: float = 0.0

@app.post("/items", status_code=201)
async def create_item(item: Item):
    """Create a new item."""
    total = item.price + item.tax
    return {
        "name": item.name,
        "price": item.price,
        "tax": item.tax,
        "total": total
    }
```

Test:

```bash
curl -X POST http://localhost:8000/items \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","price":999.99,"tax":79.99}'
```

Go to http://localhost:8000/docs, find the /items endpoint, and click "Try it out" to see the request schema.

## Cheat Sheet: FastAPI Fundamentals

### Route Decorators

```python
@app.get("/path")
@app.post("/path")
@app.put("/path/{id}")
@app.delete("/path/{id}")
@app.patch("/path/{id}")
async def my_route():
    return {}
```

### Parameter Types

```python
# Path parameter (in URL)
@app.get("/items/{item_id}")
async def get_item(item_id: int):
    ...

# Query parameter (after ?)
@app.get("/search")
async def search(q: str, limit: int = 10):
    ...

# Request body (JSON)
from pydantic import BaseModel
class Item(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create(item: Item):
    ...
```

### Status Codes

```python
@app.post("/items", status_code=201)
async def create_item(item: Item):
    ...
```

### Common Status Codes

| Code | Use |
|------|-----|
| 200 | OK (GET, PUT) |
| 201 | Created (POST) |
| 204 | No Content (DELETE) |
| 400 | Bad Request |
| 401 | Unauthorized |
| 404 | Not Found |
| 422 | Validation Error |
| 500 | Server Error |

### Running Uvicorn

```bash
uvicorn main:app --reload              # Development
uvicorn main:app --port 8000           # Custom port
uvicorn main:app --workers 4           # Multiple workers
```

## Key Takeaways

- **FastAPI is ASGI** — handles multiple requests concurrently, not just sequentially
- **Type hints are validation** — `name: str` automatically validates the input type
- **Pydantic models validate request bodies** — complex data with validation rules
- **/docs endpoint is auto-generated** — write once, document for free
- **Path parameters, query parameters, request body** — know when to use each
- **Status codes matter** — 200 for success, 201 for created, 4xx for client errors, 5xx for server errors
- **async def is required** — for non-blocking I/O (will explain in Module 2)

Now let's explore why async matters and when you must use it.
