# Module 7: FastAPI in Production

## The Analogy: Restaurant Grand Opening

When a small restaurant wants to expand:

- **Testing phase**: Chef tests recipes, trains staff, runs practice services
- **Grand opening prep**: Check restrooms, test water pressure, hire security, plan backup for rush hours
- **Opening day**: Doors open, hundreds arrive, chaos is managed with systems
- **Post-opening**: Monitor reviews, fix issues, improve processes

Moving FastAPI to production requires:

1. **Configuration** — Database, API keys, environment variables
2. **Multiple workers** — Handle high traffic
3. **Health checks** — Know if service is alive
4. **Graceful shutdown** — Don't disconnect mid-request
5. **Logging** — Track problems in production
6. **Monitoring** — Detect issues before users do

## Development vs Production

### Development

```bash
uvicorn main:app --reload
```

One worker, auto-reloads on code changes, verbose logging, debug mode.

### Production

```bash
gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --access-logfile - \
  --error-logfile - \
  --log-level info
```

Multiple workers, no auto-reload, structured logging, no debug mode.

## Configuration Management

### Problem: Hardcoded Secrets

```python
# WRONG!
DATABASE_URL = "postgresql://user:password@localhost/mydb"
SECRET_KEY = "super-secret-key"
API_KEY = "12345"

@app.post("/login")
async def login(...):
    ...
```

Anyone with access to code sees secrets. Not secure.

### Solution: Environment Variables

```python
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    # Database
    database_url: str = os.getenv("DATABASE_URL")
    
    # Security
    secret_key: str = os.getenv("SECRET_KEY")
    api_key: str = os.getenv("API_KEY")
    
    # Server
    debug: bool = os.getenv("DEBUG", "false").lower() == "true"
    workers: int = int(os.getenv("WORKERS", "4"))
    
    # Services
    redis_url: str = os.getenv("REDIS_URL", "redis://localhost:6379")
    celery_broker: str = os.getenv("CELERY_BROKER_URL", "redis://localhost:6379/0")
    
    class Config:
        env_file = ".env"  # Load from .env file

settings = Settings()
```

**.env file** (never commit to git!):

```
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/mydb
SECRET_KEY=production-secret-key-12345
API_KEY=api-key-from-vendor
DEBUG=false
WORKERS=4
REDIS_URL=redis://localhost:6379
```

**.gitignore**:

```
.env
*.log
__pycache__/
```

### Use Settings in Code

```python
from config import settings
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(settings.database_url)

@app.on_event("startup")
async def startup():
    # Validate settings on startup
    if not settings.secret_key:
        raise ValueError("SECRET_KEY not set")
```

## Health Checks: Readiness and Liveness

### Readiness Check

"Is the service ready to handle requests?"

Checks:
- Database connection: Can I query?
- Redis connection: Can I reach the broker?
- Configuration: Are all settings valid?

```python
@app.get("/ready")
async def readiness_check(db: AsyncSession = Depends(get_db)):
    """Readiness check — used by load balancers"""
    try:
        # Check database
        await db.execute(text("SELECT 1"))
        
        # Check Redis
        redis_client = await redis.from_url(settings.redis_url)
        await redis_client.ping()
        
        return {"status": "ready"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Not ready: {e}")
```

### Liveness Check

"Is the service alive (not deadlocked)?"

Simpler than readiness. Just returns OK if the process is running.

```python
@app.get("/live")
async def liveness_check():
    """Liveness check — used by Kubernetes"""
    return {"status": "alive"}
```

### How Load Balancers Use These

```
Nginx (load balancer)
├─ GET /ready → 200? Send traffic to this worker
├─ GET /ready → 503? Remove from pool
└─ Check every 10 seconds
```

If a worker's database connection dies:

```
1. New request arrives
2. Nginx checks /ready
3. Worker tries to query database
4. Database is down → 503 Service Unavailable
5. Nginx removes worker from pool
6. Next request goes to healthy worker
```

## Graceful Shutdown

### The Problem: Request Cutoff

Without proper shutdown:

```
Worker 1: Processing PDF (6 seconds remaining)
Nginx:    "Shut down worker 1"
Worker 1: [KILL - process dies]
Client:   "Connection reset by peer"
PDF:      Half processed, lost
```

### The Solution: Shutdown Signal Handler

```python
import signal
import asyncio

shutdown_event = asyncio.Event()

async def shutdown_handler():
    shutdown_event.set()

signal.signal(signal.SIGTERM, lambda *_: asyncio.create_task(shutdown_handler()))

@app.on_event("shutdown")
async def on_shutdown():
    """Wait for in-flight requests to complete"""
    print("Shutting down: waiting for in-flight requests...")
    await asyncio.sleep(2)  # Grace period
    print("Shutdown complete")
```

### Deployment: Zero-Downtime Deployment

```bash
#!/bin/bash
# deploy.sh - Zero-downtime deployment

# 1. Start new worker
docker run -d \
  -e DATABASE_URL=$DB_URL \
  -p 8001:8000 \
  myapp:latest

# 2. Wait for health check
while ! curl -f http://localhost:8001/ready; do
  sleep 1
done

# 3. Add to load balancer
docker exec nginx nginx -s reload

# 4. Wait a bit
sleep 10

# 5. Remove old worker (Nginx sends SIGTERM, waits 30s)
kill -TERM $(docker ps | grep myapp:v1 | awk '{print $1}')

# 6. Wait for graceful shutdown
sleep 35

# 7. Kill if still running
docker kill $(docker ps | grep myapp:v1 | awk '{print $1}') || true
```

## Structured Logging

### Problem: Unstructured Logs

```
INFO: 2024-01-15 14:30:00 - User alice logged in
ERROR: 2024-01-15 14:30:05 - Database error
WARNING: 2024-01-15 14:30:10 - slow request
```

Hard to parse, hard to search in production.

### Solution: JSON Logging

```python
import logging
import json
from pythonjsonlogger import jsonlogger

# Setup JSON logging
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
logHandler.setFormatter(formatter)
logger = logging.getLogger()
logger.addHandler(logHandler)
logger.setLevel(logging.INFO)

# Usage
logger.info("user_login", extra={"user_id": 5, "email": "alice@example.com"})

# Output:
# {"message": "user_login", "user_id": 5, "email": "alice@example.com", "timestamp": "2024-01-15T14:30:00Z"}
```

In production, this JSON is parsed by log aggregators (ELK, CloudWatch, etc.):

```
Filter: user_id=5 → Find all actions by user 5
Filter: status=error → Find all errors
Filter: response_time>5000 → Find slow requests
```

## Docker Deployment

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY . .

# Run with Gunicorn (production ASGI server)
CMD ["gunicorn", "main:app", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

### docker-compose.yml

```yaml
version: "3.8"

services:
  api:
    build: .
    container_name: pdf_api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@postgres:5432/mydb
      SECRET_KEY: ${SECRET_KEY}
      DEBUG: "false"
      WORKERS: 4
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - postgres
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/live"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Running in Production

```bash
# Build image
docker build -t myapp:prod .

# Start with docker-compose
docker-compose -f docker-compose.yml up -d

# Check health
docker-compose ps

# View logs
docker-compose logs -f api

# Scale workers
docker-compose up -d --scale api=3
```

## Real PDF SaaS Production Config

```python
# config.py
from pydantic import BaseSettings
import os

class Settings(BaseSettings):
    # App
    app_name: str = "PDF Processing API"
    debug: bool = os.getenv("DEBUG", "false").lower() == "true"
    
    # Database
    database_url: str = os.getenv("DATABASE_URL")
    database_echo: bool = False
    
    # Security
    secret_key: str = os.getenv("SECRET_KEY")
    access_token_expire_minutes: int = 30
    
    # Redis
    redis_url: str = os.getenv("REDIS_URL", "redis://localhost:6379/0")
    
    # Celery
    celery_broker_url: str = os.getenv("CELERY_BROKER_URL")
    celery_result_backend: str = os.getenv("CELERY_RESULT_BACKEND")
    
    # MinIO
    minio_url: str = os.getenv("MINIO_URL")
    minio_key: str = os.getenv("MINIO_KEY")
    minio_secret: str = os.getenv("MINIO_SECRET")
    
    # Logging
    log_level: str = "INFO"
    
    class Config:
        env_file = ".env"

settings = Settings()

# main.py
from fastapi import FastAPI
from config import settings
from routers import jobs, users, auth
import logging

# Setup logging
logging.basicConfig(level=settings.log_level)
logger = logging.getLogger(__name__)

app = FastAPI(
    title=settings.app_name,
    debug=settings.debug
)

# Include routers
app.include_router(auth.router)
app.include_router(users.router)
app.include_router(jobs.router)

@app.get("/health")
async def health():
    return {"status": "ok"}

@app.get("/ready")
async def readiness(db: AsyncSession = Depends(get_db)):
    try:
        await db.execute(text("SELECT 1"))
        return {"ready": True}
    except Exception as e:
        logger.error(f"Readiness check failed: {e}")
        raise HTTPException(status_code=503)

@app.on_event("startup")
async def startup():
    logger.info(f"Starting {settings.app_name}")
    if not settings.secret_key:
        raise ValueError("SECRET_KEY not set")

@app.on_event("shutdown")
async def shutdown():
    logger.info("Shutting down")
```

## Hands-On Lab

### Lab 7.1: Configuration with Environment Variables

Create `.env`:

```
DATABASE_URL=sqlite+aiosqlite:///:memory:
SECRET_KEY=test-secret
DEBUG=true
WORKERS=2
```

Create `config.py`:

```python
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    database_url: str = os.getenv("DATABASE_URL")
    secret_key: str = os.getenv("SECRET_KEY")
    debug: bool = os.getenv("DEBUG", "false").lower() == "true"
    workers: int = int(os.getenv("WORKERS", "1"))
    
    class Config:
        env_file = ".env"

settings = Settings()
```

Create `main.py`:

```python
from fastapi import FastAPI
from config import settings

app = FastAPI(debug=settings.debug)

@app.get("/config")
async def get_config():
    return {
        "debug": settings.debug,
        "workers": settings.workers,
        "database": "***" if settings.database_url else None
    }
```

Test:

```bash
uvicorn main:app --reload
curl http://localhost:8000/config
```

### Lab 7.2: Health Checks

Add to `main.py`:

```python
@app.get("/live")
async def liveness():
    return {"status": "alive"}

@app.get("/ready")
async def readiness():
    # Simulate checking database
    try:
        # In real app: await db.execute(text("SELECT 1"))
        return {"status": "ready"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Test:

```bash
curl http://localhost:8000/live
curl http://localhost:8000/ready
```

### Lab 7.3: Graceful Shutdown

Create `app_shutdown.py`:

```python
import asyncio
import signal
from fastapi import FastAPI

app = FastAPI()

processing_requests = set()

@app.get("/process/{task_id}")
async def process(task_id: int):
    processing_requests.add(task_id)
    try:
        await asyncio.sleep(5)
        return {"task_id": task_id, "status": "completed"}
    finally:
        processing_requests.discard(task_id)

@app.on_event("shutdown")
async def shutdown_event():
    print(f"Shutting down: {len(processing_requests)} requests in flight")
    while processing_requests:
        await asyncio.sleep(0.5)
    print("All requests completed, shutdown done")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Test:

```bash
# Terminal 1
python app_shutdown.py

# Terminal 2
curl http://localhost:8000/process/1

# Terminal 1: Press Ctrl+C during the request
# Should wait for request to complete, then shutdown
```

## Cheat Sheet: Production Deployment

### Environment Variables

```python
from pydantic import BaseSettings
import os

class Settings(BaseSettings):
    database_url: str = os.getenv("DATABASE_URL")
    secret_key: str = os.getenv("SECRET_KEY")
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### Health Checks

```python
@app.get("/live")
async def liveness():
    return {"status": "alive"}

@app.get("/ready")
async def readiness():
    # Check dependencies
    return {"status": "ready"}
```

### Graceful Shutdown

```python
@app.on_event("shutdown")
async def shutdown():
    # Wait for requests to complete
    pass
```

### Docker Run

```bash
docker run -d \
  -e DATABASE_URL="..." \
  -e SECRET_KEY="..." \
  -p 8000:8000 \
  myapp:latest
```

### Gunicorn (ASGI)

```bash
gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000
```

## Key Takeaways

- **Configuration via environment variables** — never hardcode secrets
- **Health checks tell load balancers if service is ready** — /ready, /live
- **Multiple workers handle concurrency** — Gunicorn + Uvicorn workers
- **Graceful shutdown prevents data loss** — wait for in-flight requests
- **JSON logging pairs with log aggregators** — parse/search in production
- **Docker + docker-compose simplify deployment** — single command starts everything
- **Zero-downtime updates possible** — health checks + rolling updates

You now have a complete FastAPI skill. Modules 1-7 covered HTTP basics through production deployment.

Next: PostgreSQL fundamentals.
