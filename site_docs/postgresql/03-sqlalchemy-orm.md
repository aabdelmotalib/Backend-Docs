# Module 3: SQLAlchemy ORM

## The Analogy: Speaking the Language of Python

Writing SQL directly:

```python
# Using raw SQL (tedious)
query = "SELECT * FROM users WHERE id = %s"
result = await db.execute(query, (1,))
user_dict = dict(result)
print(user_dict)
```

Using SQLAlchemy ORM:

```python
# Using Python objects (natural)
user = await session.get(User, 1)
print(user.email)
```

The ORM (Object-Relational Mapping) lets you work with Python objects instead of SQL strings. Tables become classes. Rows become instances.

## SQLAlchemy Models

A **model** is a Python class that represents a database table.

```python
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    name = Column(String(100))
    created_at = Column(DateTime, default=datetime.utcnow)
```

This model:
- Maps to `users` table in database
- `id` is a SERIAL PRIMARY KEY
- `email` is VARCHAR(255) UNIQUE NOT NULL
- `created_at` defaults to now

## Creating and Using Sessions

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

# Create async engine
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    echo=False
)

# Create async session factory
async_session = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# Use in code
async with async_session() as session:
    # Queries here
    user = await session.get(User, 1)
    return user
```

The session is a **gateway to the database**. All queries go through it.

## CRUD Operations

### Create (INSERT)

```python
from sqlalchemy import insert

async with async_session() as session:
    new_user = User(
        email="alice@example.com",
        name="Alice"
    )
    session.add(new_user)
    await session.commit()
    print(new_user.id)  # Auto-generated
```

Or using insert:

```python
async with async_session() as session:
    stmt = insert(User).values(
        email="bob@example.com",
        name="Bob"
    )
    result = await session.execute(stmt)
    await session.commit()
```

### Read (SELECT)

Single row by primary key:

```python
user = await session.get(User, 1)
print(user.email)
```

Using filter:

```python
from sqlalchemy import select

stmt = select(User).where(User.email == "alice@example.com")
result = await session.execute(stmt)
user = result.scalar_one_or_none()
```

All rows:

```python
stmt = select(User)
result = await session.execute(stmt)
users = result.scalars().all()
```

With ordering and limit:

```python
from sqlalchemy import desc

stmt = select(User).order_by(desc(User.created_at)).limit(10)
result = await session.execute(stmt)
recent_users = result.scalars().all()
```

### Update (UPDATE)

```python
from sqlalchemy import update

stmt = update(User).where(User.id == 1).values(name="Alice Smith")
await session.execute(stmt)
await session.commit()
```

Or modify object and commit:

```python
user = await session.get(User, 1)
user.name = "Alice Smith"
await session.commit()
```

### Delete (DELETE)

```python
user = await session.get(User, 1)
await session.delete(user)
await session.commit()
```

## Relationships: One-to-Many

A user has many subscriptions:

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    
    subscriptions = relationship("Subscription", back_populates="user")

class Subscription(Base):
    __tablename__ = "subscriptions"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    plan = Column(String(50))
    
    user = relationship("User", back_populates="subscriptions")
```

Usage:

```python
user = await session.get(User, 1)
print(user.subscriptions)  # List of Subscription objects

for sub in user.subscriptions:
    print(f"  {sub.plan}")
```

## Pydantic Integration

Convert SQLAlchemy models to Pydantic for API responses:

```python
from pydantic import BaseModel
from datetime import datetime

class UserResponse(BaseModel):
    id: int
    email: str
    name: str
    created_at: datetime
    
    class Config:
        from_attributes = True  # Read from ORM model

# In FastAPI route
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    session: AsyncSession = Depends(get_session)
):
    user = await session.get(User, user_id)
    return UserResponse.from_orm(user)
```

Or using computed fields:

```python
from pydantic import field_validator

class UserResponse(BaseModel):
    id: int
    email: str
    subscription_count: int
    
    @field_validator("subscription_count", mode="before")
    @classmethod
    def count_subs(cls, v, values):
        return len(values.get("subscriptions", []))
```

## Eager Loading: Preventing N+1 Queries

### The Problem

```python
users = await session.scalars(select(User))
for user in users:
    print(user.subscriptions)  # QUERY for each user!
```

If 100 users, this runs:
- 1 query to get users
- 100 queries to get subscriptions

Total: 101 queries. Slow!

### The Solution: joinedload

```python
from sqlalchemy.orm import joinedload

stmt = (
    select(User)
    .options(joinedload(User.subscriptions))
)
users = await session.scalars(stmt)
for user in users:
    print(user.subscriptions)  # No additional queries!
```

Now one query with a JOIN. Much faster.

## Real PDF SaaS Models

```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean, ForeignKey, Text
from sqlalchemy.orm import relationship, declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    subscriptions = relationship("Subscription", back_populates="user")

class Subscription(Base):
    __tablename__ = "subscriptions"
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    plan = Column(String(50))
    pages_per_month = Column(Integer)
    active = Column(Boolean, default=True)
    user = relationship("User", back_populates="subscriptions")
    jobs = relationship("Job", back_populates="subscription")

class Job(Base):
    __tablename__ = "jobs"
    id = Column(Integer, primary_key=True)
    subscription_id = Column(Integer, ForeignKey("subscriptions.id"), nullable=False)
    input_url = Column(String(500))
    output_format = Column(String(10))
    status = Column(String(50), default="queued")
    created_at = Column(DateTime, default=datetime.utcnow)
    subscription = relationship("Subscription", back_populates="jobs")
```

Usage in FastAPI:

```python
@app.post("/jobs", response_model=JobResponse)
async def create_job(
    job_data: JobCreate,
    current_user: int = Depends(get_current_user),
    session: AsyncSession = Depends(get_session)
):
    # Get subscription
    stmt = select(Subscription).where(
        (Subscription.id == job_data.subscription_id) &
        (Subscription.user_id == current_user)
    )
    sub = await session.scalar(stmt)
    if not sub:
        raise HTTPException(status_code=404)
    
    # Create job
    job = Job(
        subscription_id=sub.id,
        input_url=job_data.input_url,
        output_format=job_data.output_format
    )
    session.add(job)
    await session.commit()
    
    return job
```

## Hands-On Lab

### Lab 3.1: Define Models

Create `models.py`:

```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Boolean
from sqlalchemy.orm import declarative_base, relationship
from datetime import datetime

Base = declarative_base()

class Author(Base):
    __tablename__ = "authors"
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = "books"
    id = Column(Integer, primary_key=True)
    title = Column(String(255), nullable=False)
    author_id = Column(Integer, ForeignKey("authors.id"), nullable=False)
    published = Column(Boolean, default=False)
    author = relationship("Author", back_populates="books")
```

### Lab 3.2: CRUD Operations

Create `app.py`:

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import select
from models import Base, Author, Book

engine = create_async_engine("sqlite+aiosqlite:///:memory:")
async_session = sessionmaker(engine, class_=AsyncSession)

# Create tables
async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def test():
    await init_db()
    
    # Create
    async with async_session() as session:
        author = Author(name="J.K. Rowling")
        session.add(author)
        await session.commit()
        print(f"Created author ID: {author.id}")
        
        book = Book(title="Harry Potter", author_id=author.id)
        session.add(book)
        await session.commit()
    
    # Read
    async with async_session() as session:
        stmt = select(Author).where(Author.name == "J.K. Rowling")
        author = await session.scalar(stmt)
        print(f"Found: {author.name}")
        print(f"Books: {len(author.books)}")

import asyncio
asyncio.run(test())
```

## Cheat Sheet: SQLAlchemy ORM

### Define Model

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String(255))
    subscriptions = relationship("Subscription", back_populates="user")
```

### Session

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine("postgresql+asyncpg://...")
session_factory = sessionmaker(engine, class_=AsyncSession)

async with session_factory() as session:
    # Queries here
    pass
```

### CRUD

```python
# Create
user = User(email="alice@example.com")
session.add(user)
await session.commit()

# Read
user = await session.get(User, 1)

# Update
user.email = "alice2@example.com"
await session.commit()

# Delete
await session.delete(user)
await session.commit()
```

### Query

```python
from sqlalchemy import select

stmt = select(User).where(User.email == "alice@example.com")
user = await session.scalar(stmt)

users = await session.scalars(select(User))
```

## Key Takeaways

- **Models are Python classes** — define database structure in code
- **ORM handles SQL generation** — you don't write SQL strings
- **Sessions are gateways** — all queries go through them
- **Relationships are defined in code** — easier than foreign keys
- **Eager loading prevents N+1 queries** — use joinedload
- **Pydantic + SQLAlchemy** — convert ORM objects to API responses
- **Async sessions** — use with FastAPI for non-blocking queries

Module 4 teaches Alembic migrations — version control for database schema.
