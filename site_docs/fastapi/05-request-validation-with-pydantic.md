# Module 5: Request Validation with Pydantic

## The Analogy: Border Agent Checking Documents

When you cross a border, an agent checks:

- Does your passport exist?
- Is it valid?
- Are dates correct?
- Does your face match the photo?

If anything is wrong, they reject you before you enter the country.

Pydantic is that agent for your API. Before your code executes, Pydantic validates:

- Is the required field present?
- Is it the correct type?
- Is the value in the allowed range?
- Does the format match (email, URL, date)?

If validation fails, return 422 Unprocessable Entity immediately. Your code never runs.

## Why Validation Matters

Without validation:

```python
@app.post("/users")
async def create_user(user: User):
    # user.age might be "not a number"
    # user.email might be "not-an-email"
    # user.name might be a number
    # Your code crashes trying to process garbage
```

With Pydantic:

```python
from pydantic import BaseModel, EmailStr, Field

class User(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    age: int = Field(..., ge=0, le=150)
    email: EmailStr  # Validates email format

@app.post("/users")
async def create_user(user: User):
    # If we reach here, all fields are valid
    # user.name is definitely a string
    # user.age is definitely an int between 0-150
    # user.email is definitely a valid email
```

## Pydantic BaseModel Basics

### Simple Model

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
```

Usage:

```python
# Valid
item = Item(name="Book", price=19.99)
item.name   # "Book"
item.price  # 19.99

# Invalid
try:
    item = Item(name="Book", price="expensive")  # price must be float
except ValidationError as e:
    print(e)
    # price: value is not a valid float
```

### Type Support

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional, List

class Order(BaseModel):
    order_id: int
    items: List[str]
    qty: int
    created: datetime
    notes: Optional[str] = None  # Can be None
    tags: List[str] = []         # Default empty list
```

Pydantic validates each field:

- `order_id`: must be int
- `items`: must be list of strings
- `created`: must be datetime (accepts ISO string, converts automatically)
- `notes`: string or None
- `tags`: list of strings (optional, defaults to empty)

### Nested Models

```python
class Address(BaseModel):
    street: str
    city: str
    country: str

class User(BaseModel):
    name: str
    address: Address  # Nested model
    friends: List[Address] = []  # List of nested

# Usage
user_data = {
    "name": "Alice",
    "address": {
        "street": "123 Main St",
        "city": "New York",
        "country": "USA"
    },
    "friends": [...]
}
user = User(**user_data)
user.address.city  # "New York"
```

Pydantic validates the nested model too.

## Field Validation: Constraints and Rules

### Basic Constraints

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0, le=999999)  # > 0, <= 999999
    age: int = Field(..., ge=0, le=100)         # >= 0, <= 100
    quantity: int = Field(10, ge=0)             # Default 10, >= 0
    discount: float = Field(0.0, ge=0, le=1)    # 0-1 (0% to 100%)
```

Field constraints:

| Constraint | Meaning |
|-----------|---------|
| `gt` | Greater than |
| `ge` | Greater than or equal |
| `lt` | Less than |
| `le` | Less than or equal |
| `min_length` | Minimum string length |
| `max_length` | Maximum string length |
| `regex` | Must match regex pattern |

### Email Validation

```python
from pydantic import EmailStr

class User(BaseModel):
    email: EmailStr  # Validates email format

# Valid: alice@example.com, test.user+tag@gmail.co.uk
# Invalid: alice@, @example.com, alice example.com
```

### URL Validation

```python
from pydantic import HttpUrl

class Link(BaseModel):
    url: HttpUrl  # Must be valid HTTP(S) URL

# Valid: https://example.com, http://api.example.com
# Invalid: ftp://example.com, not-a-url
```

### Date and Time

```python
from datetime import datetime, date
from pydantic import BaseModel

class Event(BaseModel):
    title: str
    date: date
    start_time: datetime

# FastAPI converts ISO strings automatically
# "2024-01-15" → date(2024, 1, 15)
# "2024-01-15T14:30:00" → datetime(2024, 1, 15, 14, 30)
```

## Custom Validators

### Validator Decorator

```python
from pydantic import BaseModel, validator

class User(BaseModel):
    username: str
    password: str
    
    @validator("username")
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError("Username must be alphanumeric")
        return v
    
    @validator("password")
    def password_length(cls, v):
        if len(v) < 8:
            raise ValueError("Password must be >= 8 characters")
        return v

# Valid
User(username="alice123", password="securepass456")

# Invalid
try:
    User(username="alice@123", password="short")
except ValidationError as e:
    print(e)
    # username: Username must be alphanumeric
    # password: Password must be >= 8 characters
```

### Root Validator (Multi-Field)

Sometimes validation depends on multiple fields:

```python
from pydantic import BaseModel, root_validator

class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime
    
    @root_validator
    def check_range(cls, values):
        start = values.get("start_date")
        end = values.get("end_date")
        if start >= end:
            raise ValueError("start_date must be before end_date")
        return values

# Invalid
try:
    DateRange(
        start_date=datetime(2024, 1, 15),
        end_date=datetime(2024, 1, 10)
    )
except ValidationError:
    print("start_date must be before end_date")
```

## Request/Response Models

In production, you often have different models for:

- **Request** (what client sends): May be incomplete
- **Database** (internal): Includes id, created_at
- **Response** (what API returns): Filtered fields

### Example

```python
from pydantic import BaseModel
from datetime import datetime

# What client sends when creating a user
class UserCreate(BaseModel):
    name: str
    email: str
    password: str  # Only on create, not returned

# What's stored in database
class UserInDB(BaseModel):
    id: int
    name: str
    email: str
    password_hash: str
    created_at: datetime
    class Config:
        from_attributes = True  # Allow ORM model conversion

# What API returns (public)
class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    created_at: datetime
    class Config:
        from_attributes = True  # ORM model → Pydantic

@app.post("/users", response_model=UserResponse)
async def create_user(user: UserCreate, db: AsyncSession = Depends(get_db)):
    # user is UserCreate (name, email, password)
    # Hash password, save to DB
    db_user = UserInDB(...)
    # Response is UserResponse (id, name, email, created_at)
    # password_hash never returned to client
    return db_user
```

## Swagger Documentation from Pydantic

FastAPI reads Pydantic models and generates documentation:

```python
from pydantic import BaseModel, Field
from typing import Optional

class JobCreate(BaseModel):
    """Request body for creating a PDF job"""
    
    pdf_url: str = Field(
        ...,
        description="URL to the PDF file",
        example="https://example.com/document.pdf"
    )
    
    job_name: str = Field(
        ...,
        min_length=1,
        max_length=100,
        description="Human-readable job name",
        example="Convert to JPG"
    )
    
    page_count: Optional[int] = Field(
        None,
        description="Number of pages to process (null = all pages)",
        example=10
    )
    
    priority: int = Field(
        1,
        ge=1,
        le=5,
        description="Priority level (1=low, 5=high)",
        example=3
    )

# FastAPI generates Swagger with all descriptions
# Users see what each field is, constraints, examples
```

## Real PDF SaaS Example

```python
from pydantic import BaseModel, validator
from datetime import datetime
from typing import List

class JobCreate(BaseModel):
    """Create a new PDF processing job"""
    pdf_url: str  # URL to upload
    output_format: str  # "jpg", "png", "pdf"
    pages: Optional[str] = None  # "1,3-5" or None for all
    
    @validator("output_format")
    def validate_format(cls, v):
        if v not in ["jpg", "png", "pdf"]:
            raise ValueError(f"Format must be jpg/png/pdf, got {v}")
        return v

class JobResponse(BaseModel):
    """Job data returned to client"""
    id: int
    pdf_url: str
    output_format: str
    status: str  # "queued", "processing", "completed", "failed"
    created_at: datetime
    completed_at: Optional[datetime] = None
    
    class Config:
        from_attributes = True  # Convert SQLAlchemy to Pydantic

class JobList(BaseModel):
    """List of jobs"""
    total: int
    jobs: List[JobResponse]
    has_more: bool
```

Usage in routes:

```python
@app.post("/jobs", response_model=JobResponse, status_code=201)
async def create_job(
    job: JobCreate,
    current_user: int = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """User creates a PDF job"""
    # job is validated: pdf_url exists, format is valid
    # Create database record, queue task
    return job_response

@app.get("/jobs", response_model=JobList)
async def list_jobs(
    current_user: int = Depends(get_current_user),
    skip: int = 0,
    limit: int = 10,
    db: AsyncSession = Depends(get_db)
):
    """List user's jobs with pagination"""
    # Fetch from database
    total = await db.scalar(select(func.count(Job.id)))
    jobs = await db.scalars(
        select(Job)
        .where(Job.user_id == current_user)
        .offset(skip)
        .limit(limit)
    )
    return JobList(total=total, jobs=jobs, has_more=skip + limit < total)
```

## Hands-On Lab

### Lab 5.1: Build a Validation Model

Create `validation_app.py`:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field, validator, EmailStr
from typing import Optional

app = FastAPI()

class ProductCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0, le=999999)
    stock: int = Field(..., ge=0)
    description: Optional[str] = Field(None, max_length=1000)
    
    @validator("price")
    def price_precision(cls, v):
        # Only 2 decimal places
        if len(str(v).split(".")[-1]) > 2:
            raise ValueError("Price must have max 2 decimal places")
        return v

class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    stock: int

products_db = {}
next_id = 1

@app.post("/products", response_model=ProductResponse)
async def create_product(product: ProductCreate):
    global next_id
    db_product = {
        "id": next_id,
        **product.dict()
    }
    products_db[next_id] = db_product
    result = next_id
    next_id += 1
    return db_product

@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int):
    if product_id not in products_db:
        raise HTTPException(status_code=404)
    return products_db[product_id]
```

Test:

```bash
# Valid request
curl -X POST http://localhost:8000/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Book","price":19.99,"stock":5}'

# Invalid: negative price
curl -X POST http://localhost:8000/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Book","price":-10,"stock":5}'
# 422 Unprocessable Entity

# Invalid: missing name
curl -X POST http://localhost:8000/products \
  -H "Content-Type: application/json" \
  -d '{"price":19.99,"stock":5}'
# 422 Unprocessable Entity
```

### Lab 5.2: Nested Models

Add to `validation_app.py`:

```python
from typing import List

class OrderCreate(BaseModel):
    email: EmailStr
    products: List[int] = Field(..., min_items=1)  # At least 1 product
    
    @validator("email")
    def email_domain(cls, v):
        if not v.endswith("@example.com"):
            raise ValueError("Email must be @example.com")
        return v

@app.post("/orders")
async def create_order(order: OrderCreate):
    return {"email": order.email, "product_count": len(order.products)}
```

Test:

```bash
curl -X POST http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","products":[1,2]}'
```

## Cheat Sheet: Pydantic Models

### Basic Model

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str
```

### Constraints

```python
from pydantic import Field

class User(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    age: int = Field(..., ge=0, le=150)
    price: float = Field(..., gt=0)
```

### Special Types

```python
from pydantic import EmailStr, HttpUrl
from datetime import datetime
from typing import Optional

class Event(BaseModel):
    name: str
    email: EmailStr
    website: HttpUrl
    date: datetime
    notes: Optional[str] = None
```

### Validators

```python
from pydantic import validator

class User(BaseModel):
    username: str
    
    @validator("username")
    def alpha(cls, v):
        if not v.isalnum():
            raise ValueError("Must be alphanumeric")
        return v
```

### Config

```python
class User(BaseModel):
    name: str
    
    class Config:
        from_attributes = True  # Convert ORM to Pydantic
        json_schema_extra = {
            "example": {"name": "Alice"}
        }
```

## Key Takeaways

- **Pydantic validates before code runs** — fail fast with 422 errors
- **Type hints are validation rules** — `age: int` rejects "90 apples"
- **Field() adds constraints** — `min_length`, `ge`, `le`, `regex`
- **Validators customize logic** — email domain, password strength
- **Nested models validate recursively** — complex structures
- **Request/Response models separate concerns** — never return passwords
- **Swagger auto-generates from Pydantic** — documentation for free

Now Module 6 teaches middleware and error handling across all routes.
