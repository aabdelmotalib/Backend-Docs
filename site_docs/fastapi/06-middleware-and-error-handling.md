# Module 6: Middleware and Error Handling

## The Analogy: Hotel Front Desk

When a guest arrives at a hotel:

1. **Before checking in** (incoming middleware): Verify ID, scan for weapons, check reservations
2. **Room service** (route handler): Guest uses room, calls for room service
3. **After checkout** (outgoing middleware): Clean room, charge card, log departure

Middleware runs before and after your routes, handling cross-cutting concerns.

## What Middleware Does

Middleware intercepts requests and responses:

```
Request
   ↓
[CORS Check]
   ↓
[Authentication]
   ↓
[Logging]
   ↓
Route Handler
   ↓
[Add Headers]
   ↓
[Log Response]
   ↓
Response
```

Common uses:

1. **CORS** — Allow cross-origin requests
2. **Rate Limiting** — Prevent abuse
3. **Logging** — Log every request/response
4. **Error Handling** — Catch exceptions, return 500
5. **Security Headers** — Add security headers
6. **Request ID** — Track requests across services

## CORS: Handling Cross-Origin Requests

### The Problem

Your API is at `api.example.com`. A frontend at `app.example.com` makes a request:

```javascript
fetch('https://api.example.com/users')
```

Browser blocks it:

```
Access to XMLHttpRequest at 'https://api.example.com/users'
from origin 'https://app.example.com' has been blocked by CORS policy
```

This is a security feature. Without CORS, any website could read your API.

### The Solution: Add CORS Middleware

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com", "https://admin.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)

@app.get("/users")
async def get_users():
    return ["alice", "bob"]
```

Now requests from `app.example.com` are allowed.

### CORS in Production

```python
import os
from fastapi.middleware.cors import CORSMiddleware

# Allow only specific origins
if os.getenv("ENVIRONMENT") == "production":
    origins = ["https://app.example.com"]
else:
    origins = ["http://localhost:3000", "http://localhost:8080"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

## Custom Middleware

Create middleware using `@app.middleware`:

```python
from fastapi import FastAPI
from time import time
import logging

app = FastAPI()
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request, call_next):
    # Before route
    start = time()
    
    # Call the route
    response = await call_next(request)
    
    # After route
    duration = time() - start
    logger.info(f"{request.method} {request.url.path} - {response.status_code} - {duration:.2f}s")
    
    return response
```

### Middleware Order Matters

```python
# CORS first (changes response headers)
app.add_middleware(CORSMiddleware, ...)

# Then Custom (logs everything)
@app.middleware("http")
async def log_requests(request, call_next):
    ...
```

### Add Request ID (for Distributed Tracing)

```python
import uuid
from fastapi import FastAPI, Request

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id
    
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

@app.get("/users")
async def get_users(request: Request):
    request_id = request.state.request_id
    return {"request_id": request_id}
```

## Global Exception Handlers

### Problem: Unhandled Exceptions

When an exception occurs in a route:

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one()  # Raises NoResultFound
    return user
```

If user_id doesn't exist, `scalar_one()` raises an exception. Client gets 500 Internal Server Error. Not helpful.

### Solution: Handle Specific Exceptions

```python
from fastapi import FastAPI, HTTPException
from sqlalchemy.exc import NoResultFound

@app.exception_handler(NoResultFound)
async def no_result_handler(request, exc):
    return JSONResponse(
        status_code=404,
        content={"detail": "User not found"}
    )

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one()  # Raises NoResultFound → 404
    return user
```

### Handle All Exceptions

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def generic_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={
            "detail": "Internal server error",
            "message": str(exc) if os.getenv("ENV") == "development" else None
        }
    )
```

## Structured Error Responses

### Consistent Error Format

```python
from pydantic import BaseModel

class ErrorResponse(BaseModel):
    error: str
    detail: str
    status_code: int

@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=400,
        content={
            "error": "ValidationError",
            "detail": str(exc),
            "status_code": 400
        }
    )
```

### Application-Specific Exceptions

```python
class PDFProcessingError(Exception):
    def __init__(self, message: str, job_id: int = None):
        self.message = message
        self.job_id = job_id

@app.exception_handler(PDFProcessingError)
async def pdf_error_handler(request: Request, exc: PDFProcessingError):
    return JSONResponse(
        status_code=400,
        content={
            "error": "PDFProcessingError",
            "detail": exc.message,
            "job_id": exc.job_id
        }
    )

# Use in route
@app.post("/jobs")
async def process_job(job: JobCreate):
    if not is_valid_pdf(job.pdf_url):
        raise PDFProcessingError("Invalid PDF format", job_id=None)
    return {"status": "processing"}
```

## Request Context with Dependency Injection

### Extract Request Metadata

```python
from fastapi import Request, Depends

async def get_request_metadata(request: Request):
    return {
        "method": request.method,
        "path": request.url.path,
        "client_ip": request.client.host,
        "user_agent": request.headers.get("user-agent")
    }

@app.get("/info")
async def get_info(metadata: dict = Depends(get_request_metadata)):
    return metadata
```

### User Context

```python
async def get_current_user_context(
    current_user: int = Depends(get_current_user),
    request: Request = None
):
    return {
        "user_id": current_user,
        "request_id": request.state.request_id,
        "path": request.url.path
    }

@app.get("/dashboard")
async def dashboard(context: dict = Depends(get_current_user_context)):
    return {"user_context": context}
```

## Real PDF SaaS Error Handling

```python
from fastapi import FastAPI, HTTPException, status
from fastapi.responses import JSONResponse

app = FastAPI()

# Custom exceptions
class JobNotFoundError(Exception):
    pass

class QuotaExceededError(Exception):
    pass

# Exception handlers
@app.exception_handler(JobNotFoundError)
async def job_not_found_handler(request: Request, exc: JobNotFoundError):
    return JSONResponse(
        status_code=status.HTTP_404_NOT_FOUND,
        content={"detail": "Job not found"}
    )

@app.exception_handler(QuotaExceededError)
async def quota_exceeded_handler(request: Request, exc: QuotaExceededError):
    return JSONResponse(
        status_code=status.HTTP_429_TOO_MANY_REQUESTS,
        content={"detail": "Monthly PDF processing quota exceeded"}
    )

# Middleware for logging
@app.middleware("http")
async def log_all(request: Request, call_next):
    start_time = time.time()
    try:
        response = await call_next(request)
        duration = time.time() - start_time
        logger.info(
            f"{request.method} {request.url.path} {response.status_code} {duration:.2f}s"
        )
        return response
    except Exception as e:
        duration = time.time() - start_time
        logger.error(f"{request.method} {request.url.path} failed after {duration:.2f}s: {e}")
        raise

# Routes
@app.post("/jobs", response_model=JobResponse, status_code=201)
async def create_job(
    job: JobCreate,
    current_user: int = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Create PDF processing job"""
    # Check quota
    used = await get_user_quota_used(current_user, db)
    if used >= 1000:  # 1000 pages per month
        raise QuotaExceededError("Monthly quota exceeded")
    
    # Create and return
    return job_response

@app.get("/jobs/{job_id}", response_model=JobResponse)
async def get_job(
    job_id: int,
    current_user: int = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Get job details"""
    result = await db.execute(
        select(Job).where((Job.id == job_id) & (Job.user_id == current_user))
    )
    job = result.scalar_one_or_none()
    
    if not job:
        raise JobNotFoundError()
    
    return job
```

## Hands-On Lab

### Lab 6.1: Add CORS Middleware

Create `cors_app.py`:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"]
)

@app.get("/items")
async def get_items():
    return [{"id": 1, "name": "Item 1"}]

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Test from browser console:

```javascript
// From localhost:3000
fetch('http://localhost:8000/items')
  .then(r => r.json())
  .then(d => console.log(d))
```

### Lab 6.2: Custom Logging Middleware

Add to `cors_app.py`:

```python
from time import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request, call_next):
    start = time()
    path = request.url.path
    method = request.method
    
    response = await call_next(request)
    
    duration = time() - start
    logger.info(
        f"{method:6s} {path:20s} {response.status_code} {duration:.3f}s"
    )
    return response
```

### Lab 6.3: Exception Handler

Add to `cors_app.py`:

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse

class ItemNotFoundError(Exception):
    pass

@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(request, exc):
    return JSONResponse(
        status_code=404,
        content={"detail": "Item not found"}
    )

items_db = {1: {"id": 1, "name": "Item 1"}}

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id not in items_db:
        raise ItemNotFoundError()
    return items_db[item_id]
```

Test:

```bash
curl http://localhost:8000/items/1    # 200 OK
curl http://localhost:8000/items/999  # 404 Item not found
```

## Cheat Sheet: Middleware and Errors

### CORS Setup

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["GET", "POST"],
    allow_headers=["*"]
)
```

### Custom Middleware

```python
@app.middleware("http")
async def my_middleware(request, call_next):
    # Before route
    response = await call_next(request)
    # After route
    return response
```

### Exception Handler

```python
@app.exception_handler(CustomError)
async def custom_handler(request, exc):
    return JSONResponse(
        status_code=400,
        content={"detail": str(exc)}
    )
```

### Request ID

```python
import uuid

@app.middleware("http")
async def add_request_id(request, call_next):
    request.state.request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request.state.request_id
    return response
```

## Key Takeaways

- **Middleware runs for every request** — CORS, logging, auth
- **Middleware → Route → Middleware** — like a sandwich
- **CORS allows cross-origin requests** — configure trusted origins
- **Custom middleware for logging, tracing, security**
- **Exception handlers catch errors** — return consistent responses
- **Structured errors help debugging** — include error codes
- **Request context available** — client IP, user agent, headers

Now Module 7 teaches production deployment Best practices: configuration, health checks, graceful shutdown.
