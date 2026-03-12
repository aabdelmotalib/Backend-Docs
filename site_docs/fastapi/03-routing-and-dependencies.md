# Module 3: Routing and Dependency Injection

## The Analogy: Organizing a Restaurant

When a restaurant is small, one chef handles everything. But as it grows:

- **Small**: Chef takes order, cooks, serves (all in one person)
- **Large**: Front desk takes order → passes to appetizer station → passes to entree station → passes to dessert station → serves customer

Each station (module) has its responsibility. Orders flow through them.

In FastAPI:

- **Small app**: One file with all routes
- **Growing app**: Use `APIRouter` to organize routes into stations (modules)
- **Large app**: Use `Depends()` for shared logic (database, authentication)

## From Monolithic to Modular Routes

### The Problem: One File Gets Messy

```python
# main.py (600 lines, unmaintainable)

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    ...

@app.post("/users")
async def create_user(user: UserCreate):
    ...

@app.put("/users/{user_id}")
async def update_user(user_id: int, user: UserUpdate):
    ...

@app.delete("/users/{user_id}")
async def delete_user(user_id: int):
    ...

@app.get("/subscriptions/{subscription_id}")
async def get_subscription(subscription_id: int):
    ...

# ... 100 more routes ...
```

### The Solution: APIRouter

Create separate files for each resource:

```
src/
├── main.py
├── routers/
│   ├── users.py
│   ├── subscriptions.py
│   └── health.py
└── models.py
```

**routers/users.py**

```python
from fastapi import APIRouter, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

@router.post("")
async def create_user(name: str):
    return {"name": name, "created": True}

@router.put("/{user_id}")
async def update_user(user_id: int, name: str):
    return {"user_id": user_id, "name": name}

@router.delete("/{user_id}")
async def delete_user(user_id: int):
    return {"user_id": user_id, "deleted": True}
```

**routers/subscriptions.py**

```python
from fastapi import APIRouter

router = APIRouter(prefix="/subscriptions", tags=["subscriptions"])

@router.get("/{subscription_id}")
async def get_subscription(subscription_id: int):
    return {"subscription_id": subscription_id}

@router.post("")
async def create_subscription(user_id: int, plan: str):
    return {"user_id": user_id, "plan": plan}
```

**main.py**

```python
from fastapi import FastAPI
from routers import users, subscriptions, health

app = FastAPI()

# Include routers
app.include_router(users.router)
app.include_router(subscriptions.router)
app.include_router(health.router)

@app.get("/")
async def root():
    return {"message": "API"}
```

Now:
- `GET /users/1` calls `users.router`
- `POST /subscriptions` calls `subscriptions.router`
- Each route lives in its own file
- Easy to maintain and test

## Dependency Injection: The Depends() Pattern

### The Problem: Repeated Code

Imagine every route needs a database session:

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    async with async_session() as session:  # Repeated every route
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

@app.get("/subscriptions/{subscription_id}")
async def get_subscription(subscription_id: int):
    async with async_session() as session:  # Repeated every route
        result = await session.execute(
            select(Subscription).where(Subscription.id == subscription_id)
        )
        return result.scalar_one_or_none()
```

This is DRY (Don't Repeat Yourself) violation.

### The Solution: Depends()

Create a dependency function:

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session
        # Automatically cleanup after route completes
```

Use it with `Depends()`:

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()

@app.get("/subscriptions/{subscription_id}")
async def get_subscription(
    subscription_id: int, 
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(
        select(Subscription).where(Subscription.id == subscription_id)
    )
    return result.scalar_one_or_none()
```

### How Depends() Works

When a route has a parameter with `Depends()`:

```
1. FastAPI sees: db: AsyncSession = Depends(get_db)
2. Calls get_db() before the route runs
3. Passes the result to the route as `db` parameter
4. Route executes
5. Cleanup code in get_db() runs (context manager exit)
```

## Common Dependencies

### Database Session

```python
async def get_db():
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    ...
```

### Current User (Authentication)

```python
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    # Validate token, return user object
    payload = jwt.decode(token, SECRET_KEY)
    user_id = payload.get("user_id")
    # Fetch from database
    return user

@app.get("/profile")
async def get_profile(current_user: User = Depends(get_current_user)):
    return {"user_id": current_user.id, "email": current_user.email}
```

### Request-Specific Config

```python
async def get_config(request: Request):
    return {
        "user_agent": request.headers.get("user-agent"),
        "client_ip": request.client.host
    }

@app.get("/info")
async def get_info(config: dict = Depends(get_config)):
    return config
```

## Dependency Chains: Dependencies Depending on Dependencies

A dependency can itself have dependencies:

```python
async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    payload = jwt.decode(token, SECRET_KEY)
    # Use db to fetch user
    result = await db.execute(
        select(User).where(User.id == payload["user_id"])
    )
    return result.scalar_one()

async def get_user_subscriptions(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
) -> List[Subscription]:
    # Current user already authenticated
    result = await db.execute(
        select(Subscription).where(Subscription.user_id == current_user.id)
    )
    return result.scalars().all()

@app.get("/subscriptions")
async def list_subscriptions(
    subscriptions: List[Subscription] = Depends(get_user_subscriptions)
):
    return subscriptions
```

The dependency tree:

```
list_subscriptions
└── get_user_subscriptions
    ├── get_current_user
    │   ├── oauth2_scheme (from security)
    │   └── get_db
    └── get_db (reused!)
```

FastAPI caches dependencies in a request scope, so `get_db()` is called once and reused.

## Path and Query Parameter Extraction with Query(), Path()

Sometimes you need validation or documentation:

```python
from fastapi import Query, Path

@app.get("/search")
async def search(
    q: str = Query(..., min_length=1, max_length=100, description="Search term"),
    limit: int = Query(10, ge=1, le=100, description="Result limit"),
    offset: int = Query(0, ge=0, description="Result offset")
):
    return {"q": q, "limit": limit, "offset": offset}

@app.get("/items/{item_id}")
async def get_item(
    item_id: int = Path(..., gt=0, description="Item ID must be positive")
):
    return {"item_id": item_id}
```

The `Path()` and `Query()` objects add:
- Validation rules (`min_length`, `ge`, `le`, `gt`, `lt`)
- Documentation in Swagger
- Default values
- Required/optional markers (`...` means required)

## Real Project Structure

```
src/
├── main.py                 # FastAPI app creation, router includes
├── config.py               # Settings (database URL, API keys)
├── database.py             # Database setup, session factory
├── schemas.py              # Pydantic models (request/response)
├── models.py               # SQLAlchemy ORM models
├── dependencies.py         # Shared Depends() functions
├── routers/
│   ├── __init__.py
│   ├── health.py           # Health checks
│   ├── jobs.py             # PDF job routes
│   ├── users.py            # User CRUD
│   └── subscriptions.py     # Subscription routes
└── services/
    ├── __init__.py
    ├── job_service.py      # Job business logic
    └── email_service.py    # Email notifications
```

## Hands-On Lab

### Lab 3.1: Split Routes into APIRouter

Create the structure:

```bash
mkdir -p fastapi_app/src/routers
cd fastapi_app
```

**src/routers/health.py**

```python
from fastapi import APIRouter

router = APIRouter(tags=["health"])

@router.get("/health")
async def health():
    return {"status": "ok"}

@router.get("/ready")
async def ready():
    return {"ready": True}
```

**src/routers/items.py**

```python
from fastapi import APIRouter

router = APIRouter(prefix="/items", tags=["items"])

@router.get("/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id, "name": f"Item {item_id}"}

@router.post("")
async def create_item(name: str, price: float):
    return {"name": name, "price": price, "created": True}
```

**src/main.py**

```python
from fastapi import FastAPI
from routers import health, items

app = FastAPI()

app.include_router(health.router)
app.include_router(items.router)

@app.get("/")
async def root():
    return {"message": "API"}
```

Test:

```bash
uvicorn src.main:app --reload
curl http://localhost:8000/health
curl http://localhost:8000/items/1
curl -X POST "http://localhost:8000/items?name=Book&price=15.99"
```

### Lab 3.2: Add Dependency Injection

**src/dependencies.py**

```python
from fastapi import Depends

async def get_query_log(q: str = None):
    """Log query parameters"""
    return {"query": q, "logged": True}

async def get_api_key(api_key: str = Depends(...)):
    """Validate API key"""
    if api_key != "secret":
        raise HTTPException(status_code=401, detail="Invalid API key")
    return api_key
```

**src/routers/items.py** (updated)

```python
from fastapi import APIRouter, Depends
from dependencies import get_query_log

router = APIRouter(prefix="/items", tags=["items"])

@router.get("/{item_id}")
async def get_item(
    item_id: int,
    log: dict = Depends(get_query_log)
):
    return {"item_id": item_id, "log": log}
```

### Lab 3.3: Database Session Dependency

**src/database.py**

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "sqlite+aiosqlite:///:memory:",  # In-memory SQLite
    echo=False
)

async_session = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db():
    async with async_session() as session:
        yield session
```

**src/main.py** (updated)

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from database import get_db
from routers import health, items

app = FastAPI()

app.include_router(health.router)
app.include_router(items.router)

@app.get("/db-test")
async def db_test(db: AsyncSession = Depends(get_db)):
    # Session is available, automatically cleaned up
    return {"db": str(db), "connected": True}
```

## Cheat Sheet: Routers and Dependencies

### APIRouter Basic

```python
from fastapi import APIRouter

router = APIRouter(prefix="/items", tags=["items"])

@router.get("/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id}

# In main.py:
app.include_router(router)
```

### Dependency with Yield

```python
from fastapi import Depends

async def get_resource():
    resource = create_resource()  # Setup
    yield resource                # Use
    cleanup(resource)             # Teardown

@app.get("/")
async def endpoint(resource = Depends(get_resource)):
    return resource
```

### Dependency Chain

```python
async def get_db():
    yield db

async def get_current_user(db = Depends(get_db)):
    return user

@app.get("/")
async def endpoint(user = Depends(get_current_user)):
    return user
```

### Query and Path Validation

```python
from fastapi import Query, Path

@app.get("/items/{item_id}")
async def get_item(
    item_id: int = Path(..., gt=0),
    q: str = Query(None, min_length=1)
):
    return {"item_id": item_id, "q": q}
```

## Key Takeaways

- **APIRouter** organizes routes into logical modules
- **Depends()** injects dependencies and avoids repeated code
- **Dependencies can have dependencies** — form dependency trees
- **Yield in dependencies** ensures cleanup runs automatically
- **Query() and Path()** add validation rules and documentation
- **Database session should be a dependency** — scoped to request lifetime

Now Module 4 teaches JWT authentication using dependencies.
