# Module 4: Authentication and JWT

## The Analogy: Passport Control

When you enter a country:

1. **Registration**: You apply for a passport. Border agent verifies identity. Issues passport with an expiration date.
2. **Entry**: You show passport. Agent scans it. Checks expiration. If valid, you enter.
3. **Problem**: If passport expires mid-visit, you're stuck. You need to renew.

JWT tokens work the same way:

1. **Login**: User provides username/password. Server verifies. Issues JWT token with expiration.
2. **Access**: User sends JWT with each request. Server verifies signature. If valid, grant access.
3. **Expiration**: Token expires after 1 hour. User must login again.

## Why Authentication Matters

Without authentication, anyone can:

- Access other users' data
- Start PDF processing jobs without paying
- Modify user settings
- Cancel subscriptions

With authentication:

- Users login with credentials
- Each request includes a token proving identity
- Only their own data is accessible

## JWT (JSON Web Tokens): The Format

A JWT has three parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI1IiwiZXhwIjoxNzAxMDUwMDAwfQ.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
│                          │                                           │
Header                  Payload                                   Signature
```

### Header

Base64-encoded JSON:

```json
{
  "alg": "HS256",   // Algorithm: HMAC SHA-256
  "typ": "JWT"      // Type: JWT
}
```

### Payload

Base64-encoded JSON (the actual data):

```json
{
  "sub": "5",           // Subject: user ID
  "email": "alice@example.com",
  "exp": 1701050000    // Expiration: Unix timestamp
}
```

### Signature

Created by signing the first two parts with a secret key:

```
HMAC-SHA256(
  base64(header) + "." + base64(payload),
  "your-secret-key"
)
```

If someone modifies the payload, the signature becomes invalid.

## The Login Flow

```
User                           Server
  │                              │
  ├─── POST /login              │
  │     {"user": "alice",        │
  │      "password": "pass123"}  │
  │                              │
  │                    Verify credentials ✓
  │                    Hash password matches
  │                              │
  │◀─ 200 OK                     │
  │    {access_token: "...",     │
  │     token_type: "bearer"}    │
  │                              │
  ├─── GET /profile              │
  │     Header: Authorization: Bearer <token>
  │                              │
  │                    Verify signature ✓
  │                    Check expiration ✓
  │                    Extract user_id = 5 ✓
  │                              │
  │◀─ 200 OK                     │
  │    {user_id: 5, ...}        │
```

## Implementing JWT in FastAPI

### Step 1: Install Dependencies

```bash
pip install python-jose cryptography passlib[bcrypt] python-multipart
```

### Step 2: Create Dependencies

**dependencies.py**

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import bcrypt

SECRET_KEY = "your-secret-key-change-this"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

def verify_password(plain: str, hashed: str) -> bool:
    return bcrypt.checkpw(plain.encode(), hashed.encode())

def create_access_token(user_id: int, expires_delta: timedelta = None) -> str:
    to_encode = {"sub": str(user_id)}
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)) -> int:
    """Validate JWT token, return user_id"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return int(user_id)
    except JWTError:
        raise HTTPException(status_code=401, detail="Could not validate credentials")
```

### Step 3: Create Login Endpoint

**routers/auth.py**

```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from pydantic import BaseModel
from dependencies import (
    create_access_token,
    verify_password,
    hash_password
)

router = APIRouter(tags=["auth"])

# Mock user database
USERS_DB = {
    "alice": {
        "user_id": 1,
        "email": "alice@example.com",
        "password_hash": hash_password("password123")
    }
}

class Token(BaseModel):
    access_token: str
    token_type: str

@router.post("/login", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login endpoint: returns JWT token"""
    user = USERS_DB.get(form_data.username)
    
    if not user or not verify_password(form_data.password, user["password_hash"]):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"}
        )
    
    access_token = create_access_token(user["user_id"])
    return {"access_token": access_token, "token_type": "bearer"}
```

### Step 4: Protect Routes

```python
from fastapi import FastAPI
from dependencies import get_current_user
from routers import auth

app = FastAPI()
app.include_router(auth.router)

@app.get("/profile")
async def get_profile(current_user: int = Depends(get_current_user)):
    """Only accessible with valid JWT"""
    return {"user_id": current_user, "email": "alice@example.com"}

@app.post("/logout")
async def logout(current_user: int = Depends(get_current_user)):
    """Protected endpoint: user_id is available"""
    return {"message": f"User {current_user} logged out"}
```

## Bcrypt: Secure Password Hashing

Never store passwords in plain text. Always hash them.

### Why Bcrypt

- **Salted**: Each password gets a random salt
- **Slow**: Intentionally slow to prevent brute-force attacks
- **Irreversible**: You cannot decrypt hashes

### Usage

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Hash a password
hashed = pwd_context.hash("password123")
# Result: "$2b$12$..."

# Verify a password
is_correct = pwd_context.verify("password123", hashed)  # True
is_wrong = pwd_context.verify("wrongpassword", hashed)   # False
```

Never do:

```python
password = "password123"  # WRONG! Anyone with database access sees the password
```

## Access Token vs Refresh Token

### Access Token

- Short-lived (15 min - 1 hour)
- Used for every request
- If leaked, attacker has limited time window

### Refresh Token

- Long-lived (7 days - 30 days)
- Stored securely (HTTP-only cookie)
- Used to get a new access token when it expires

### Flow

```
1. User logs in
   ├─ Server issues access_token (15 min) + refresh_token (7 days)
   ├─ access_token in memory (JavaScript)
   └─ refresh_token in HTTP-only cookie (cannot be stolen by JavaScript)

2. User makes requests
   ├─ Send access_token in Authorization header
   └─ Server validates, processes request

3. After 15 minutes
   ├─ access_token expires
   ├─ Client sends refresh_token to /refresh endpoint
   └─ Server issues new access_token

4. After 7 days
   ├─ refresh_token expires
   └─ User must login again
```

Implement refresh:

```python
@router.post("/refresh")
async def refresh_token(refresh_token: str):
    """Get a new access token using refresh token"""
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid refresh token")
    
    access_token = create_access_token(user_id)
    return {"access_token": access_token, "token_type": "bearer"}
```

## Real Database Integration

In production, fetch user from database:

```python
from sqlalchemy import select
from models import User

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = int(payload.get("sub"))
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    user = result.scalar_one_or_none()
    
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    
    return user
```

## Hands-On Lab

### Lab 4.1: Implement JWT Login

Create `auth_app.py`:

```python
from datetime import datetime, timedelta
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel

SECRET_KEY = "test-secret-key-12345"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

app = FastAPI()

# Mock users
USERS = {
    "alice": {
        "user_id": 1,
        "password": pwd_context.hash("password123"),
        "email": "alice@example.com"
    }
}

class Token(BaseModel):
    access_token: str
    token_type: str

def create_access_token(user_id: int):
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode = {"sub": str(user_id), "exp": expire}
    encoded = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = int(payload.get("sub"))
    except JWTError:
        raise HTTPException(status_code=401)
    return user_id

@app.post("/login", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = USERS.get(form_data.username)
    if not user or not pwd_context.verify(form_data.password, user["password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    token = create_access_token(user["user_id"])
    return {"access_token": token, "token_type": "bearer"}

@app.get("/profile")
async def profile(user_id: int = Depends(get_current_user)):
    user = list(USERS.values())[user_id - 1]
    return {"user_id": user_id, "email": user["email"]}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Test:

```bash
# Login
curl -X POST http://localhost:8000/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice&password=password123"

# Response:
# {"access_token":"eyJ...","token_type":"bearer"}

# Use token to access protected endpoint
TOKEN="eyJ..."
curl http://localhost:8000/profile \
  -H "Authorization: Bearer $TOKEN"
```

### Lab 4.2: Test Token Expiration

Modify `ACCESS_TOKEN_EXPIRE_MINUTES = 0.1` (6 seconds)

```bash
# Login and get token
TOKEN=$(curl -s -X POST http://localhost:8000/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice&password=password123" | jq -r '.access_token')

# Use immediately
curl http://localhost:8000/profile -H "Authorization: Bearer $TOKEN"
# 200 OK

# Wait 10 seconds
sleep 10

# Try again
curl http://localhost:8000/profile -H "Authorization: Bearer $TOKEN"
# 401 Unauthorized (token expired)
```

## Cheat Sheet: JWT Auth

### Create Token

```python
from jose import jwt
from datetime import datetime, timedelta

SECRET_KEY = "secret"
expire = datetime.utcnow() + timedelta(minutes=30)
token = jwt.encode(
    {"sub": "user_id", "exp": expire},
    SECRET_KEY,
    algorithm="HS256"
)
```

### Verify Token

```python
from jose import jwt, JWTError

try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    user_id = payload.get("sub")
except JWTError:
    # Invalid or expired
    raise HTTPException(status_code=401)
```

### Hash Password

```python
from passlib.context import CryptContext

pwd = CryptContext(schemes=["bcrypt"])
hashed = pwd.hash("password")
is_valid = pwd.verify("password", hashed)
```

### Login Endpoint

```python
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # Verify user
    token = create_access_token(user_id)
    return {"access_token": token, "token_type": "bearer"}
```

### Protected Route

```python
@app.get("/profile")
async def profile(user_id: int = Depends(get_current_user)):
    return {"user_id": user_id}
```

## Key Takeaways

- **Never store plain passwords** — always hash with bcrypt
- **JWT = signed token with expiration** — cannot be forged
- **Access tokens are short-lived** — 15 min to 1 hour
- **Refresh tokens are long-lived** — get new access token
- **Validate signature and expiration** — every protected request
- **OAuth2 + bcrypt** — standard for secure authentication
- **Dependencies make auth reusable** — apply to any route

Now Module 5 teaches Pydantic validation to catch invalid data before processing.
