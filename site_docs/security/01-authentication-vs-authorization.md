# Module 1: Authentication vs Authorization

## The Analogy: The Airplane Journey

**Authentication**: "Are you really John Smith?"
- Passport check: "This passport has your photo and ID number. You are John Smith."

**Authorization**: "Are you allowed on this flight?"
- Boarding pass check: "You have a ticket for Flight 123. You can board."

Two different things. You can be authenticated (proven to be John) but not authorized (don't have a boarding pass).

## Authentication: Proving Your Identity

**Authentication** answers: "Who are you?"

Mechanisms:
- Username + password (login page)
- JWT token (stateless, API header)
- OAuth2 (login with Google/Apple)
- Multi-factor authentication (password + phone code)

Your API doesn't store passwords. It stores:
1. Username
2. Hashed password (bcrypt)
3. Email

When user logs in:
1. Client sends username + password
2. Server hashes password, checks if matches stored hash
3. If match: generate JWT token
4. Return token to client

Client uses token in all future requests:
```
GET /api/documents
Authorization: Bearer eyJhbGc...
```

## Authorization: Proving Your Permissions

**Authorization** answers: "What are you allowed to do?"

Examples:
- Free tier: 10 conversions/month
- Pro tier: 1000 conversions/month
- Admin: can see all users, delete anything

Your API has three dependencies for this:

### Dependency 1: require_user (Authentication)

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

def get_current_user(credentials = Depends(security)):
    token = credentials.credentials
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401)
        return {"user_id": int(user_id)}
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/api/documents")
async def list_documents(current_user = Depends(get_current_user)):
    # This endpoint requires authentication
    user_id = current_user["user_id"]
    docs = db.query(Document).filter_by(user_id=user_id).all()
    return docs
```

### Dependency 2: require_subscription (Authorization)

```python
def require_active_subscription(current_user = Depends(get_current_user)):
    user = db.query(User).filter_by(id=current_user["user_id"]).first()
    
    if not user or user.subscription_status != "active":
        raise HTTPException(
            status_code=403,
            detail="Active subscription required"
        )
    
    return user

@app.post("/api/documents/convert")
async def convert_document(
    file: UploadFile,
    user = Depends(require_active_subscription)
):
    # This endpoint requires both authentication AND a subscription
    # Only users with active subscriptions can use this
    result = process_file(file)
    return result
```

### Dependency 3: require_admin (Authorization)

```python
def require_admin(current_user = Depends(get_current_user)):
    user = db.query(User).filter_by(id=current_user["user_id"]).first()
    
    if not user or user.role != "admin":
        raise HTTPException(
            status_code=403,
            detail="Admin access required"
        )
    
    return user

@app.post("/api/admin/users/{user_id}/ban")
async def ban_user(user_id: int, admin = Depends(require_admin)):
    # Only admins can ban users
    user = db.query(User).filter_by(id=user_id).first()
    user.banned = True
    db.commit()
    return {"status": "user banned"}
```

## Session-Based vs Token-Based Auth

### Session-Based

```
Client login
→ Server creates session (in-memory or database)
→ Server sends session_id cookie
→ Client includes cookie in all requests
→ Server looks up session to verify identity
```

Stateful: Server stores session for each user. Doesn't scale (multiple servers need shared session storage).

### Token-Based (JWT)

```
Client login
→ Server generates JWT token (signed data, cannot be forged)
→ Server sends token
→ Client includes token in Authorization header
→ Server verifies token signature (no database lookup needed)
```

Stateless: Server doesn't store anything. Just verifies the token. Scales infinitely.

JWT token:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiI1IiwiZXhwIjoxNjA4NDA3MTAwfQ
.qwerty...
```

Decoded:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "sub": "5",        // user_id
  "exp": 1608407100  // expiration
}
```

Only the server knows the secret key. If anyone tries to modify the token, the signature no longer matches.

## OAuth2 Overview

OAuth2allows users to login with existing accounts (Google, Apple, GitHub, etc.).

Flow:
```
1. User clicks "Login with Google"
2. Browser redirects to Google login
3. User logs into Google
4. Google redirects back to your app with auth code
5. Your backend exchanges auth code for access token
6. Your backend verifies token with Google
7. You create your own JWT token
8. User is logged in
```

Advantages:
- Users don't create new passwords
- Your app doesn't store passwords
- Works with existing accounts

In this project: not needed yet. Focus on JWT first.

## Hands-On Lab

### Lab 1.1: Write Authn + Authz Endpoints

```python
from fastapi import FastAPI, Depends, HTTPException
from passlib.context import CryptContext
import jwt
from datetime import datetime, timedelta

app = FastAPI()
SECRET_KEY = "your-secret-key-change-this"
ALGORITHM = "HS256"

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Simulated user database
users_db = {
    1: {"username": "alice", "email": "alice@example.com", "hashed_password": pwd_context.hash("password123"), "role": "user"},
    2: {"username": "bob", "email": "bob@example.com", "hashed_password": pwd_context.hash("password456"), "role": "admin"}
}

def get_current_user(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401)
        return int(user_id)
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_admin(user_id: int = Depends(get_current_user)):
    user = users_db.get(user_id)
    if not user or user["role"] != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return user_id

@app.post("/login")
def login(username: str, password: str):
    # Find user
    user_id = None
    for uid, user_data in users_db.items():
        if user_data["username"] == username:
            user_id = uid
            break
    
    if not user_id or not pwd_context.verify(password, users_db[user_id]["hashed_password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Generate token
    exp = datetime.utcnow() + timedelta(hours=24)
    token = jwt.encode({"sub": str(user_id), "exp": exp}, SECRET_KEY, algorithm=ALGORITHM)
    
    return {"access_token": token, "token_type": "bearer"}

@app.get("/api/documents")
def list_documents(user_id: int = Depends(get_current_user)):
    # Requires authentication
    return {"documents": []}

@app.post("/api/admin/users/ban")
def ban_user(user_id: int, target_user_id: int, admin: int = Depends(require_admin)):
    # Requires authentication AND admin role
    return {"status": "user banned"}
```

### Lab 1.2: Test with curl

```bash
# Login
curl -X POST http://localhost:8000/login \
  -d "username=alice&password=password123"
# {"access_token": "eyJhbGc..."}

# Use token to access protected endpoint
curl http://localhost:8000/api/documents \
  -H "Authorization: Bearer eyJhbGc..."
# {"documents": []}

# Try to access admin endpoint as regular user
curl -X POST http://localhost:8000/api/admin/users/ban \
  -H "Authorization: Bearer eyJhbGc..."
# 403 Forbidden: Admin access required

# Login as admin
curl -X POST http://localhost:8000/login \
  -d "username=bob&password=password456"
# {"access_token": "eyJhbGc..."}

# Access admin endpoint
curl -X POST http://localhost:8000/api/admin/users/ban \
  -H "Authorization: Bearer eyJhbGc..."
# {"status": "user banned"}
```

## Cheat Sheet: Authentication and Authorization

### Authentication (Proving Identity)

```python
# Generate token on login
exp = datetime.utcnow() + timedelta(hours=24)
token = jwt.encode({"sub": str(user_id), "exp": exp}, SECRET_KEY, algorithm="HS256")

# Verify token on each request
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
user_id = payload["sub"]
```

### Authorization (Checking Permissions)

```python
def require_user(token: str):
    user_id = jwt_decode(token)
    return user_id

def require_subscription(user_id: int):
    user = db.query(User).get(user_id)
    if user.subscription_status != "active":
        raise HTTPException(status_code=403)
    return user

def require_admin(user_id: int):
    user = db.query(User).get(user_id)
    if user.role != "admin":
        raise HTTPException(status_code=403)
    return user

@app.post("/api/admin/action")
async def admin_action(admin = Depends(require_admin)):
    ...
```

### Status Codes

```
200 OK                  = Success
401 Unauthorized        = Missing or invalid authentication
403 Forbidden           = Authenticated but not authorized (missing permission)
```

## Key Takeaways

- **Authentication** = proving who you are (login)
- **Authorization** = proving what you're allowed to do (permissions)
- **JWT = stateless tokens**, no server storage needed
- **Three dependencies**: get_current_user, require_subscription, require_admin
- **401 = authentication failed**, 403 = authorization failed
- **Never store passwords in plain text**, always use bcrypt
- **Token expiration** prevents old tokens from being used forever

Module 2 teaches encryption — protecting data before and during transmission.
