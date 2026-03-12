# Module 1: Architecture Overview

## Complete System Diagram

```mermaid
graph TB
    User["👤 User<br/>Browser"]
    DNS["🌐 DNS<br/>pdf.example.com"]
    
    subgraph "Internet"
        TLS["🔒 TLS/HTTPS<br/>Port 443"]
    end
    
    subgraph "Server (Hetzner VPS, CX21)"
        Nginx["🔌 Nginx<br/>Reverse Proxy<br/>Port 80/443"]
        
        subgraph "Docker Network"
            FastAPI["⚡ FastAPI<br/>Web API<br/>Port 8000"]
            CeleryWorker["🔄 Celery<br/>Task Worker<br/>Queue Consumer"]
            CeleryBeat["⏰ Celery Beat<br/>Scheduler<br/>Session Sync"]
            
            PostgreSQL["🗄️ PostgreSQL<br/>Truth Database<br/>Port 5432"]
            Redis["⚡ Redis<br/>Cache & Queue<br/>Port 6379"]
            MinIO["📦 MinIO<br/>File Storage<br/>Port 9000"]
            ClamAV["🛡️ ClamAV<br/>Antivirus Scan"]
            LibreOffice["📄 LibreOffice<br/>PDF Conversion"]
        end
    end
    
    ExternalAPIs["🌐 External APIs"]
    Paymob["💳 Paymob<br/>Payment Provider"]
    GitHub["🔀 GitHub<br/>CI/CD"]
    
    User -->|"1. DNS lookup"| DNS
    DNS -->|"Returns IP"| Nginx
    User -->|"2. HTTPS request<br/>JWT in header"| TLS
    TLS --> Nginx
    Nginx -->|"3. Proxy to"| FastAPI
    
    FastAPI -->|"4a. Query/Insert"| PostgreSQL
    FastAPI -->|"4b. Session check<br/>Rate limit"| Redis
    FastAPI -->|"4c. Upload file"| MinIO
    FastAPI -->|"4d. Scan file"| ClamAV
    FastAPI -->|"4e. Queue task<br/>delays.delay()"| Redis
    
    Redis -->|"5. Task ready"| CeleryWorker
    CeleryWorker -->|"6a. Download file"| MinIO
    CeleryWorker -->|"6b. Lock acquired"| Redis
    CeleryWorker -->|"6c. Convert PDF"| LibreOffice
    CeleryWorker -->|"6d. Upload result"| MinIO
    CeleryWorker -->|"6e. Update job"| PostgreSQL
    CeleryWorker -->|"6f. Update session"| Redis
    
    CeleryBeat -->|"7a. Every minute"| PostgreSQL
    CeleryBeat -->|"7b. Sync sessions"| Redis
    
    FastAPI -->|"8. Initiate payment"| Paymob
    Paymob -->|"9. Webhook callback"| FastAPI
    FastAPI -->|"10. Activate subscription<br/>Create Session"| PostgreSQL
    FastAPI -->|"11. Store session"| Redis
    
    FastAPI -->|"Response 200 OK"| Nginx
    Nginx -->|"HTML/JSON"| User
    
    GitHub -->|"Push new code"| Nginx
    
    style FastAPI fill:#4CAF50
    style CeleryWorker fill:#FF9800
    style CeleryBeat fill:#FF9800
    style PostgreSQL fill:#2196F3
    style Redis fill:#FFC107
    style MinIO fill:#9C27B0
    style ClamAV fill:#F44336
    style LibreOffice fill:#E91E63
    style Nginx fill:#607D8B
    style TLS fill:#00BCD4
```

## The 8 Core Services

### 1. **Nginx** (Reverse Proxy)
**What**: Traffic router for the entire system.

**Handles**:
- TLS/HTTPS termination (decrypts port 443)
- HTTP → HTTPS redirect
- Rate limiting (prevent brute force)
- Static file serving (gzipped)
- Proxies dynamic requests to FastAPI

**Lives**: Port 80 (HTTP), 443 (HTTPS)

**Why not just FastAPI on port 443?**
- FastAPI doesn't handle TLS efficiently
- Nginx is battle-tested, optimized for serving
- Separation of concerns (web server vs app server)

---

### 2. **FastAPI** (Web Application)
**What**: Your business logic.

**Handles**:
- `POST /auth/login` → Validate credentials, return JWT
- `POST /upload` → Accept file, scan, queue conversion
- `GET /jobs/{id}/status` → Return job status + presigned URL
- `POST /payments/initiate` → Call Paymob API
- `POST /payments/webhook` → Receive payment confirmation
- Rate limiting, CORS, input validation

**Lives**: Port 8000

**Why FastAPI?**
- Fast (Starlette is async)
- Type hints (Pydantic validation)
- Auto-generated OpenAPI docs
- Production-ready (Uvicorn ASGI server)

---

### 3. **PostgreSQL** (Truth Database)
**What**: Single source of truth for all persistent data.

**Stores**:
- Users (id, email, password_hash, subscription_id)
- Subscriptions (user_id, plan, is_active, expires_at)
- Files (user_id, filename, size, status, uploaded_at)
- Jobs (file_id, status, error, started_at, completed_at)
- Payments (user_id, amount, status, paymob_ref, created_at)

**Lives**: Port 5432 (internal)

**Why PostgreSQL?**
- ACID guarantees (money is involved)
- Relationships matter (users → subscriptions → files)
- Full-text search (search uploaded files)
- JSON support (metadata)

---

### 4. **Redis** (Speed Layer)
**What**: Fast cache + message broker.

**Stores**:
- **Sessions**: `session:{user_id}:start`, `session:{user_id}:files` (60-min TTL)
- **Rate limits**: `ratelimit:ip:{ip}:count` (auto-reset hourly)
- **JWT blacklist**: `jwt:blacklist:{token_hash}` (on logout)
- **Task queue**: `celery` queues (FIFO)
- **Locks**: `lock:conversion:{job_id}` (distributed)

**Lives**: Port 6379 (internal)

**Why Redis?**
- In-memory (subsecond response)
- TTL built-in (sessions auto-expire)
- Pub/Sub (real-time notifications)
- Atomic operations (transactions)

---

### 5. **Celery Worker** (Background Tasks)
**What**: Executes long-running tasks without blocking the user.

**Handles**:
- PDF conversion (5-30 seconds, blocks on LibreOffice)
- Image thumbnail generation
- Email sending
- Database cleanup
- Retries failed tasks

**Lives**: Inside Docker, consumes from Redis queue

**Why Celery?**
- User doesn't wait for 30-second conversion
- Can run multiple workers (parallel processing)
- Automatic retries (task fails? Retry 3 times)
- Dead letter queue (failed tasks inspectable)

---

### 6. **Celery Beat** (Scheduler)
**What**: Runs recurring tasks on a schedule.

**Schedule**:
- Every 1 minute: Sync Redis sessions to PostgreSQL (durability)
- Every 24 hours: Delete expired sessions
- Every 7 days: Cleanup old files (free plan limit)

**Why Celery Beat?**
- Distributed (survives application restart)
- Lightweight (one task per minute)
- Easy to add/remove scheduled jobs

---

### 7. **MinIO** (File Storage)
**What**: Object storage (like S3, but local).

**Stores**:
- Input PDFs: `input-files/user-123/uuid.pdf`
- Converted images: `output-files/user-123/uuid/page-1.png`
- Temporary files: `temp/uuid.txt` (auto-cleanup)

**Lives**: Port 9000 (internal)

**Why MinIO?**
- S3-compatible (easy migration to AWS S3)
- No local filesystem (scales horizontally)
- Built-in versioning
- Presigned URLs (users download directly)

---

### 8. **Support Services**

#### ClamAV (Antivirus)
Scans uploaded files for malware. Blocks if infected.

#### LibreOffice (Conversion)
Converts PDF → PNG/JPEG using headless mode. Isolated in separate container to prevent crashes.

---

## Data Flow Map

```
┌─────────────────────────────────────────────────────┐
│                  Data Residency                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  PostgreSQL (Persistent Truth)                      │
│  ├─ Users                                           │
│  ├─ Subscriptions                                   │
│  ├─ Files                                           │
│  ├─ Jobs                                            │
│  └─ Payments                                        │
│                                                      │
│  Redis (Cache & Queue, TTL-based)                   │
│  ├─ Active sessions (60 min)                        │
│  ├─ Rate limits (1 hour)                            │
│  ├─ JWT blacklist (token lifetime)                  │
│  ├─ Task queue (picked up by workers)               │
│  └─ Locks (conversion in progress)                  │
│                                                      │
│  MinIO (Object Storage)                             │
│  ├─ Input PDFs (raw uploads)                        │
│  ├─ Output images (converted results)               │
│  └─ Temporary working files                         │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## Request Flow Map

```
                        www.pdf-platform.com
                              │
                    ┌─────────┼─────────┐
                    ↓         ↓         ↓
              GET /    POST /upload  GET /jobs/{id}
            (Frontend) (Form Data)  (Poll Status)
                │         │          │
                ├─→ Nginx ←┤          │
                │         │          │
                ↓         ↓          ↓
         ┌────────────────────────────────────┐
         │        FastAPI Routes               │
         ├────────────────────────────────────┤
         │ /auth/login          → db query    │
         │ /auth/logout         → redis del   │
         │ /upload              → db + redis  │
         │ /jobs/{id}/status    → db query    │
         │ /payments/initiate   → paymob api  │
         │ /payments/webhook    → db update   │
         │ /health              → ok (200)    │
         └────────────────────────────────────┘
```

---

## Why Each Tool Was Chosen

| Tool | Alternative | Why We Chose It |
|---|---|---|
| **FastAPI** | Django, Flask, Node.js | Fastest Python web framework, type hints, auto OpenAPI docs |
| **PostgreSQL** | MongoDB, MySQL | ACID, relationships, full-text search, JSON support |
| **Redis** | Memcached, RabbitMQ | In-memory + broker + pub/sub all in one, Celery integrates seamlessly |
| **Celery** | APScheduler, Huey | Industry standard, retries, dead letter queues, remote workers |
| **MinIO** | Local filesystem, FTP | S3-compatible (easy AWS migration), scalable, no filesystem limits |
| **Nginx** | HAProxy, Caddy | Lightweight, proven, excellent TLS support |
| **LibreOffice** | ImageMagick, Ghostscript | PDF → images, free, headless mode available |
| **ClamAV** | AVG, McAfee | Free, open-source, community updated definitions |

---

## Cost Breakdown (Hetzner VPS)

```
Hetzner CX21 VPS:        €8.50/month
├─ 4 CPU cores
├─ 8GB RAM
├─ 160GB NVMe SSD
└─ Unlimited traffic

Total:                    €8.50/month (~$10 USD)

This covers:
✓ Nginx + FastAPI
✓ PostgreSQL + Redis
✓ Celery Worker + Beat
✓ MinIO for up to 1TB files
✓ LibreOffice + ClamAV
```

Running on dedicated AWS would cost $200-500/month (if scaling). Hetzner is extremely cost-effective.

---

## Next: Everything in Motion

Module 2 shows a real HTTP request from browser to response — every step.
