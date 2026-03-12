# Module 3: Common Attack Vectors

## The OWASP Top 10

OWASP = Open Web Application Security Project. Their top 10 list of vulnerabilities:

1. Broken Authentication
2. Broken Access Control
3. Injection (SQL, command, etc.)
4. Insecure Deserialization
5. Broken Access Control
6. Security Misconfiguration
7. Cross-Site Scripting (XSS)
8. Insecure Deserialization
9. Using Components with Known Vulnerabilities
10. Insufficient Logging & Monitoring

This module covers the big ones relevant to your platform.

## SQL Injection: The Classic

Attacker manipulates SQL queries via user input.

### Vulnerable Code

```python
# ❌ NEVER DO THIS
user_email = request.query_params.get("email")
query = f"SELECT * FROM users WHERE email = '{user_email}'"
result = db.execute(query)
```

Input: `admin@example.com' OR '1'='1`
Query becomes:
```sql
SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
```

`'1'='1'` is always true. Returns all users, not just admin.

Worse: Input: `admin@example.com'; DROP TABLE users; --`
```sql
SELECT * FROM users WHERE email = 'admin@example.com'; DROP TABLE users; --'
```

Deletes the entire users table. Disaster.

### Secure Code

```python
# ✅ RIGHT: Parameterized query
user_email = request.query_params.get("email")
result = db.query(User).filter(User.email == user_email).first()

# Under the hood:
# query = "SELECT * FROM users WHERE email = %s"
# execute(query, params=[user_email])
```

Parameterized queries: the SQL structure is fixed, user input is data only. Cannot break the query structure.

SQLAlchemy ORM uses parameterized queries by default. Raw SQL needs `query_params`:

```python
# ❌ VULNERABLE
query = f"SELECT * FROM users WHERE id = {user_id}"

# ✅ SECURE
from sqlalchemy import text
query = text("SELECT * FROM users WHERE id = :user_id")
result = db.execute(query, {"user_id": user_id})
```

## XSS: Cross-Site Scripting

Attacker injects JavaScript into your page.

### Example

User comment: `<script>alert('hacked')</script>`

Without protection:
```html
<!-- Renders without escaping -->
<div class="comment">
  <script>alert('hacked')</script>  <!-- RUNS! -->
</div>
```

With protection (React JSX):
```jsx
const comment = "<script>alert('hacked')</script>";
return <div className="comment">{comment}</div>;
// Renders as escaped text, script doesn't run
```

### In FastAPI/Template Rendering

```python
# Template (Jinja2)
<div>{{ user_comment }}</div>

# To escape automatically, use:
<div>{{ user_comment | escape }}</div>
```

Or use Pydantic models and return JSON. The browser handles escaping:

```python
@app.get("/api/comments/{id}")
def get_comment(id: int):
    comment = db.query(Comment).get(id)
    return {"text": comment.text}  # JSON is always safe
```

Browser receives:
```json
{"text": "<script>alert('hacked')</script>"}
```

React renders this as text, not HTML. Safe.

## CSRF: Cross-Site Request Forgery

Attacker tricks you into making requests to your bank without your knowledge.

### Example

You're logged into your bank at `bank.com`.
You visit attacker.com malicious site.

That site has:
```html
<img src="https://bank.com/api/transfer?amount=1000&to=attacker_id" />
```

Your browser, authenticated to bank.com, makes this request. $1000 transferred.

You didn't click anything. The request happened via hidden `<img>` tag.

### Protection: SameSite Cookies

Modern browsers support SameSite attribute:

```
Set-Cookie: session=abc123; SameSite=Strict
```

Meanings:
- `Strict`: Only send cookie if user is on the same domain (safest)
- `Lax`: Send if user navigates to your site, but not from frames/CORS
- `None`: Send everywhere (requires Secure flag, HTTPS only)

Nginx/FastAPI configuration:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    allow_origins=["https://example.com"],  # Specific origin
)

# Or in response
response.set_cookie("session", value, samesite="strict")
```

### Protection: CSRF Tokens

Send a unique token with each form. Verify it matches.

```html
<form method="POST" action="/api/transfer">
  <input type="hidden" name="csrf_token" value="unique_token_abc123" />
  <input type="text" name="amount" />
  <button type="submit">Transfer</button>
</form>
```

Backend verifies token matches session:

```python
@app.post("/api/transfer")
def transfer(amount: int, csrf_token: str):
    session = get_session()
    if csrf_token != session.csrf_token:
        raise HTTPException(status_code=403, detail="Invalid CSRF token")
    
    # Process transfer
```

Attacker cannot know the CSRF token. The hidden form is useless.

## Brute-Force Attacks: Trying many passwords

Attacker tries thousands of passwords per second.

### Without Protection

```python
@app.post("/api/auth/login")
def login(username: str, password: str):
    user = db.query(User).filter_by(username=username).first()
    if pwd_context.verify(password, user.hashed_password):
        return {"token": generate_jwt()}
    else:
        raise HTTPException(status_code=401, detail="Invalid credentials")
```

Attacker can call this endpoint 1000x/second. 1 million tries per day.

### Protection: Rate Limiting

Nginx rate limiting (from networking module):

```nginx
limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;

location /api/auth/login {
    limit_req zone=auth burst=10 nodelay;
    proxy_pass http://api_backend;
}
```

5 requests per minute per IP = max 7200 tries/day per IP. With bcrypt (0.5s per hash), attacker gets ~1 crack per second. Even 10,000 IPs would take 27 years.

Also: use strong passwords. With bcrypt and rate limiting, brute-force is impractical.

## Path Traversal: ../../../etc/passwd

Attacker tries to traverse directory structure.

### Vulnerable Code

```python
@app.get("/download/{filename}")
def download_file(filename: str):
    # Attacker requests:
    # /download/../../../etc/passwd
    filepath = f"/app/uploads/{filename}"
    return FileResponse(filepath)

# But this resolves to:
# /app/uploads/../../../etc/passwd → /etc/passwd (system file!)
```

### Solution: UUID Keys, Whitelist Extensions

```python
import uuid

@app.get("/download/{file_id}")
def download_file(file_id: str):
    # file_id is a UUID stored in database
    file_record = db.query(File).filter_by(id=file_id).first()
    if not file_record:
        raise HTTPException(status_code=404)
    
    # Get actual filepath from database
    filepath = file_record.storage_path
    # /app/uploads/3e4f7a8c-1234.pdf (safe UUID, not user-controlled)
    
    return FileResponse(filepath)
```

Also validate extension:

```python
ALLOWED_EXTENSIONS = {'.pdf', '.doc', '.docx', '.txt'}

if not any(filename.endswith(ext) for ext in ALLOWED_EXTENSIONS):
    raise HTTPException(status_code=400, detail="Invalid file type")
```

## SSRF: Server-Side Request Forgery

Attacker makes your server fetch malicious URLs.

### Vulnerable Code

```python
@app.post("/api/import")
def import_from_url(url: str):
    # Attacker provides: url=http://localhost:6379 (Redis)
    response = requests.get(url)
    # Your server connects to Redis!
    # Attacker can read internal configuration
    return response.text
```

### Solution: Whitelist URLs

```python
from urllib.parse import urlparse

ALLOWED_DOMAINS = ["cdn.example.com", "storage.example.com"]

@app.post("/api/import")
def import_from_url(url: str):
    parsed = urlparse(url)
    
    if parsed.hostname not in ALLOWED_DOMAINS:
        raise HTTPException(status_code=400, detail="URL not allowed")
    
    if url.startswith("http://localhost") or url.startswith("http://127.0.0.1"):
        raise HTTPException(status_code=400, detail="Local URLs not allowed")
    
    response = requests.get(url, timeout=5)
    return response.text
```

## Attack Summary

| Attack | Vulnerability | Prevention |
|--------|---------------|------------|
| SQL Injection | Raw SQL concatenation | Parameterized queries (SQLAlchemy) |
| XSS | Unescaped HTML rendering | JSX escaping, template auto-escape |
| CSRF | Hidden requests with cookies | SameSite cookies, CSRF tokens |
| Brute-Force | No rate limiting | Nginx rate limiting + bcrypt |
| Path Traversal | User-controlled filepath | UUID keys, whitelist extensions |
| SSRF | Fetch any URL | Whitelist domains, block localhost |

## Hands-On Lab

### Lab 3.1: Demonstrate SQL Injection

```python
# ❌ VULNERABLE
from sqlalchemy import text

user_email = "admin@example.com' OR '1'='1"
query = f"SELECT * FROM users WHERE email = '{user_email}'"
# Returns ALL users!

# ✅ FIX: Parameterized query
query = text("SELECT * FROM users WHERE email = :email")
result = db.execute(query, {"email": user_email})
# Returns only users matching that email
```

### Lab 3.2: CSRF Token Validation

```python
import secrets

@app.get("/api/form")
def get_form():
    token = secrets.token_urlsafe(32)
    request.session["csrf_token"] = token  # Store in session
    return {"csrf_token": token}

@app.post("/api/transfer")
def transfer(amount: int, csrf_token: str):
    session_token = request.session.get("csrf_token")
    
    if csrf_token != session_token:
        raise HTTPException(status_code=403, detail="CSRF validation failed")
    
    # Safe to process
    return {"status": "transferred"}
```

## Cheat Sheet: Attack Prevention

```python
# SQL Injection
query = text("SELECT * FROM users WHERE id = :id")
db.execute(query, {"id": user_id})

# XSS
# React: {user_input} is auto-escaped
# Jinja2: {{ user_input | escape }}

# CSRF
response.set_cookie("session", value, samesite="strict")

# Brute-Force
# Nginx: limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m

# Path Traversal
# Use database ID, not filename
filepath = db.query(File).filter_by(id=file_id).first().path

# SSRF
if not urlparse(url).hostname in ALLOWED_DOMAINS:
    raise HTTPException(status_code=400)
```

## Key Takeaways

- **SQL Injection**: Use parameterized queries, never concatenate
- **XSS**: JSX and template escaping automatically prevent it
- **CSRF**: SameSite cookies and CSRF tokens
- **Brute-Force**: Rate limiting + slow hashing (bcrypt)
- **Path Traversal**: UUID keys, database lookups
- **SSRF**: Whitelist allowed URLs
- **Validate input** on every attack vector
- **Trust nothing** from users

Module 4 teaches file upload security — the riskiest attack surface.
