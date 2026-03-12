# Module 2: HTTP and HTTPS

## The Analogy: Mail with and Without Sealing

**HTTP** (unencrypted):
- You write a postcard
- Anyone who handles it in transit can read it
- Fast (no seal/unseal overhead)
- Insecure (ISPs, government, hackers can see content)

**HTTPS** (encrypted):
- You put letter in a sealed envelope
- Only the recipient can open it
- Slightly slower (seal/unseal takes time)
- Secure (content is private)

Today, HTTPS is mandatory. HTTP is only acceptable for testing. Real APIs use HTTPS exclusively.

## HTTP Request Anatomy

Every HTTP request has this structure:

```
GET /api/documents/123 HTTP/1.1
Host: api.example.com
User-Agent: curl/7.64.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json
Accept: application/json
X-Request-ID: req-2024-001
Cookie: session_id=abc123def456

```

Breaking it down:

### Request Line

```
GET /api/documents/123 HTTP/1.1
^   ^                   ^
|   |                   |
Method  URI           Version
```

### Method: What Operation?

```
GET     = Read (retrieve data, no side effects)
POST    = Create (send data to server)
PUT     = Replace (overwrite entire resource)
PATCH   = Update (modify part of resource)
DELETE  = Delete (remove resource)
HEAD    = Like GET but no response body (just headers)
OPTIONS = What methods are allowed?
```

Safe methods (no side effects): GET, HEAD, OPTIONS
Idempotent methods (same result if repeated): GET, PUT, DELETE, HEAD, OPTIONS
Not idempotent: POST (each POST creates a new record)

### URI: Where?

```
/api/documents/123
/api/payments/webhook
/api/health
```

Typically starts with `/api/` for APIs (vs `/static/` for files, `/` for HTML pages).

### Headers: Metadata

```
Host: api.example.com
# Which server (important for reverse proxies, virtual hosting)

Authorization: Bearer eyJhbGc...
# Who is making this request (JWT token in this case)

Content-Type: application/json
# Format of the request body

Accept: application/json
# Format we want in the response

X-Request-ID: req-2024-001
# Unique ID for tracking this request through logs

Cookie: session_id=abc123def456
# Session cookie (if using session-based auth)

User-Agent: curl/7.64.1
# Which client made the request

Accept-Encoding: gzip, deflate
# Compression formats we can handle
```

### Body: The Payload

```json
{
  "name": "Monthly Report",
  "created_at": "2024-03-12T10:00:00Z"
}
```

Only for POST, PUT, PATCH. GET/DELETE typically have no body.

## HTTP Response Anatomy

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 245
Set-Cookie: session_id=xyz789; Path=/; HttpOnly; Secure
Cache-Control: private, max-age=3600
X-Request-ID: req-2024-001

{
  "id": 123,
  "name": "Monthly Report",
  "status": "completed"
}
```

### Status Line

```
HTTP/1.1 200 OK
^        ^   ^
|        |   |
Version  Code Message
```

### Status Codes: The Language of REST

**2xx: Success**
- `200 OK`: Request succeeded, response body follows
- `201 Created`: Resource created successfully (POST)
- `204 No Content`: Request succeeded, no response body

**4xx: Client Error**
- `400 Bad Request`: Malformed request (bad JSON, missing required field)
- `401 Unauthorized`: Missing or invalid authentication (no token, expired token)
- `403 Forbidden`: Authenticated but not allowed (has token, but insufficient permissions)
- `404 Not Found`: Resource does not exist
- `422 Unprocessable Entity`: Validation failed (email format wrong, user_id does not exist)
- `429 Too Many Requests`: Rate limited (too many requests per minute)

**5xx: Server Error**
- `500 Internal Server Error`: Bug in code, unhandled exception
- `502 Bad Gateway`: Nginx received no response from FastAPI (crashed?)
- `503 Service Unavailable`: Server is down for maintenance
- `504 Gateway Timeout`: Nginx waited too long for FastAPI response

### Response Headers

```
Content-Type: application/json
# Format of response body

Content-Length: 245
# Size of response body in bytes

Set-Cookie: session_id=xyz789; Path=/; HttpOnly; Secure
# Send cookie to client
# HttpOnly = cannot be accessed by JavaScript (prevents XSS theft)
# Secure = only send over HTTPS

Cache-Control: private, max-age=3600
# How long to cache
# private = only the client can cache, not proxies
# max-age=3600 = cache for 1 hour

X-Request-ID: req-2024-001
# Echo back the request ID for correlation

ETag: "abc123"
# Hash of content; client can use with If-None-Match to check if changed
```

## Real Example: FastAPI Endpoint

```python
from fastapi import FastAPI, HTTPException, status
from fastapi.responses import JSONResponse

app = FastAPI()

@app.post("/api/documents")
async def create_document(name: str, description: str):
    """
    POST /api/documents HTTP/1.1
    (request body)
    """
    
    # Validate input
    if not name or len(name) < 3:
        # 422: validation error
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            detail="Name must be at least 3 characters"
        )
    
    # Create in database
    document = Document(name=name, description=description)
    db.add(document)
    db.commit()
    
    # 201: Created with Location header
    return JSONResponse(
        status_code=status.HTTP_201_CREATED,
        content={"id": document.id, "name": document.name},
        headers={"Location": f"/api/documents/{document.id}"}
    )

@app.get("/api/documents/{doc_id}")
async def get_document(doc_id: int):
    """GET /api/documents/123 HTTP/1.1"""
    document = db.query(Document).filter_by(id=doc_id).first()
    
    if not document:
        # 404: not found
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Document {doc_id} not found"
        )
    
    # 200: OK with body
    return document

@app.delete("/api/documents/{doc_id}")
async def delete_document(doc_id: int):
    """DELETE /api/documents/123 HTTP/1.1"""
    document = db.query(Document).filter_by(id=doc_id).first()
    
    if not document:
        raise HTTPException(status_code=404)
    
    db.delete(document)
    db.commit()
    
    # 204: No Content (deleted successfully, nothing to return)
    return None  # FastAPI returns 204 for None
```

## HTTP/1.1 vs HTTP/2

### HTTP/1.1

- One request at a time per connection
- If you need 10 images, load them sequentially (or open multiple connections)
- Slow for many small requests

### HTTP/2

- Multiplexing: many requests over one connection simultaneously
- Server push: server can send resources before client asks
- Header compression: reduces bandwidth
- All modern browsers and APIs support it

Functionally identical from the API perspective. HTTP/2 is just faster.

## HTTPS: Encrypted HTTP

HTTPS wraps HTTP in TLS encryption.

### The Handshake

1. Client connects to port 443
2. Server says: "Here's my certificate, signed by Let's Encrypt"
3. Client verifies the signature (did Let's Encrypt really sign this?)
4. Both use asymmetric crypto to agree on a symmetric key
5. From now on, all data is encrypted with that key

### Server Certificate

A certificate proves the server is who they claim to be.

```
Certificate for: api.example.com
Issued by: Let's Encrypt
Valid from: 2024-01-15
Valid until: 2024-04-15 (90 days)
Public key: [2048-bit RSA public key]
```

Let's Encrypt provides free certificates, valid for 90 days. Your server automatically renews them.

### Why HTTPS Matters

Without HTTPS:
- ISP sees every request you make
- Coffee shop WiFi reads your API keys
- Government monitors traffic
- Man-in-the-middle can modify responses

With HTTPS:
- Only the client and server can read the data
- Attacker cannot modify responses
- Eavesdropper cannot steal tokens

## REST API Design

REST = Representational State Transfer. A set of conventions.

### Resources, not Actions

**❌ Bad (RPC style)**
```
/api/createUser
/api/deleteUser?id=5
/api/updateUserEmail
```

**✅ Good (REST style)**
```
POST   /api/users               # Create user
GET    /api/users/5             # Get user 5
PUT    /api/users/5             # Replace user 5 entirely
PATCH  /api/users/5             # Update some fields of user 5
DELETE /api/users/5             # Delete user 5
```

The resource is `/api/users/5`. The method (GET, POST, etc.) describes the operation.

### Headers vs URL vs Body

**Headers**: Metadata
```
Authorization: Bearer token
X-Request-ID: req-001
```

**URL path**: Resource identifier
```
/api/users/5  # resource is user with id 5
```

**URL query parameters**: Filtering
```
/api/documents?status=completed&limit=10
```

**Body**: Input data (for POST/PUT/PATCH)
```json
{
  "name": "New name",
  "email": "user@example.com"
}
```

## Hands-On Lab

### Lab 2.1: Inspect HTTP with curl -v

```bash
# Make a request and see everything
curl -v https://api.example.com/documents

# Output shows:
# * Request header:
# > GET /documents HTTP/1.1
# > Host: api.example.com
# > User-Agent: curl/7.64.1
#
# * Response header:
# < HTTP/1.1 200 OK
# < Content-Type: application/json
# < Set-Cookie: session=abc; HttpOnly; Secure
# <
# * Response body:
# [{"id": 1, "name": "Doc 1"}]
```

### Lab 2.2: POST with Body

```bash
curl -X POST https://api.example.com/documents \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token" \
  -d '{"name": "My Document"}'
```

### Lab 2.3: Check Response Status

```bash
# Get only status code
curl -s -o /dev/null -w "%{http_code}\n" https://api.example.com/

# 200 = OK
# 404 = Not found
# 503 = Server down
```

## Cheat Sheet: HTTP Status Codes and Headers

### Codes Quick Reference

```
2xx Success
  200 OK
  201 Created
  204 No Content

4xx Client Error
  400 Bad Request
  401 Unauthorized (missing/invalid auth)
  403 Forbidden (auth OK, permissions denied)
  404 Not Found
  422 Validation Error
  429 Too Many Requests (rate limited)

5xx Server Error
  500 Internal Server Error
  502 Bad Gateway
  503 Service Unavailable
  504 Gateway Timeout
```

### Important Headers

```
Content-Type: application/json        # Format of body
Authorization: Bearer TOKEN            # Auth token
Cookie: session=abc123                # Session cookie
Set-Cookie: session=xyz; HttpOnly     # Set cookie from server
X-Request-ID: req-001                 # Unique request ID
Cache-Control: max-age=3600           # Cache for 1 hour
```

### curl Cheat Sheet

```bash
curl https://example.com              # GET
curl -X POST https://example.com      # POST
curl -H "Header: value" ...           # Add header
curl -d '{"key": "value"}' ...        # Add body (-H Content-Type needed)
curl -v https://example.com           # Verbose (see all headers)
curl -i https://example.com           # Include response headers
curl -s https://example.com           # Silent (no progress)
curl -o /dev/null -w "%{http_code}"   # Just status code
```

## Key Takeaways

- **HTTP request** = method, URI, headers, body
- **HTTP response** = status code, headers, body
- **Status codes** tell you what happened (2xx success, 4xx client error, 5xx server error)
- **HTTPS** = HTTP + TLS encryption for privacy
- **REST** = organize APIs around resources, use HTTP methods for operations
- **Always use HTTPS** in production

Module 3 teaches DNS — the system that turns `example.com` into IP addresses.
