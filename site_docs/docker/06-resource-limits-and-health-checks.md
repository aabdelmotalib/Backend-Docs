# Module 6: Resource Limits and Health Checks

## The Analogy: The Leaking Roof

Imagine you rent an apartment building to many tenants. One tenant:

- Never cleans, causing a stink that permeates the whole building
- Leaves the heat on 24/7, making the whole floor hot
- Leaves the bathtub running, flooding into the unit below

**Without rules** (resource limits), one bad tenant destroys the whole building. **With rules** (limits), that tenant is isolated. The leak affects only their unit.

Docker containers are the same. One container with a memory leak can crash your entire server, taking down your database and API.

## Why Resource Limits Are Critical in Production

### Scenario: The Memory Leak

Your PDF processing worker has a memory leak. It starts at 200MB but grows over time:

- 5 minutes: 400MB
- 10 minutes: 600MB
- 15 minutes: 800MB
- 20 minutes: 1000MB (out of memory)

**Without limits:**
- The container gets killed by the OS (OOM kill)
- ... but not before consuming 2GB of server RAM
- This causes the database and other containers to slow down
- Users experience timeouts and errors

**With a 512MB limit:**
- The container is killed at 512MB
- Other containers continue running normally
- You see the error in logs and can debug the leak

## Setting Resource Limits in Docker Compose

### Memory Limits

```yaml
services:
  worker:
    image: myworker:latest
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

- `limits.memory: 512M` — Hard limit. Kill the container if it exceeds this.
- `reservations.memory: 256M` — Soft guarantee. Docker tries to reserve this much.

**Reservation vs Limit:**

- **Reservation**: The amount of RAM Docker guarantees for the container. Other containers can still use more if available.
- **Limit**: The absolute maximum. The container gets killed if it exceeds this.

### CPU Limits

```yaml
services:
  api:
    image: myapi:latest
    deploy:
      resources:
        limits:
          cpus: '2'
        reservations:
          cpus: '1.5'
```

- `cpus: '2'` — Max 2 CPU cores
- `cpus: '0.5'` — Half a core
- `cpus: '1.5'` — 1.5 cores

**How CPU limits work:**
If the server has 4 physical CPU cores and you set `cpus: 1`, the container is throttled to never use more than 25% of the processor.

### Real-World Example

```yaml
version: '3.9'

services:
  api:
    image: fastapi:latest
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1'
        reservations:
          memory: 256M
          cpus: '0.5'

  worker:
    image: pyramid-worker:latest
    deploy:
      resources:
        limits:
          memory: 1G  # PDF processing needs more memory
          cpus: '2'
        reservations:
          memory: 600M
          cpus: '1'

  postgres:
    image: postgres:15
    deploy:
      resources:
        limits:
          memory: 2G  # Databases can benefit from more RAM
          cpus: '2'
        reservations:
          memory: 1G
          cpus: '1'

  redis:
    image: redis:7
    deploy:
      resources:
        limits:
          memory: 256M  # Cache, so less memory needed
          cpus: '0.5'
        reservations:
          memory: 128M
          cpus: '0.25'
```

## What Happens When a Container Exceeds Memory Limit

When you `docker run -m 256m myapp` and the app tries to allocate more than 256MB:

```
$ docker run -m 256m ubuntu:24.04 bash -c "python -c 'import os; os.system(\"cat /dev/zero\")'"

Killed  # Abrupt termination, exit code 137 (OOM kill)
```

The message "Killed" is actually the OS kernel terminating the process. The container dies, and the orchestrator (Docker Compose, Kubernetes) counts it as a container failure.

!!! danger
    When a container is OOM-killed, it doesn't get to shut down gracefully. It just disappears. Use restart policies to recover:
    ```yaml
    restart: on-failure
    ```

## Health Checks: Is the Container Actually Healthy?

A **health check** is a command Docker runs periodically to test if your service is actually working.

### Basic Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

| Parameter | Meaning |
|-----------|---------|
| `--interval=30s` | Check every 30 seconds |
| `--timeout=3s` | Wait up to 3 seconds for the check to complete |
| `--start-period=5s` | Give the app 5 seconds to start before checking |
| `--retries=3` | Mark unhealthy after 3 consecutive failures |

### Health Check States

- `starting` → Container just started, health check hasn't passed yet
- `healthy` → Health check passed
- `unhealthy` → Health check failed too many times

You can see the state with:

```bash
docker ps
```

Output:

```
STATUS              PORTS
Up 2 minutes (healthy)  8000/tcp
```

### Example: PostgreSQL Health Check

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --start-period=40s --retries=5 \
  CMD pg_isready -U postgres || exit 1
```

Why `start_period=40s`? PostgreSQL takes ~30 seconds to initialize on first startup.

### Example: Redis Health Check

```yaml
redis:
  image: redis:7
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 3s
    retries: 3
```

### Example: FastAPI Health Check

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1
```

Inside your FastAPI app:

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/health")
async def health():
    # You can add database checks here
    return JSONResponse({"status": "ok"})
```

## Using Health Checks with depends_on

**Critical for startup order:**

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 3s
      retries: 5

  api:
    build: ./api
    depends_on:
      postgres:
        condition: service_healthy  # Wait for health check to pass
```

Without `service_healthy`:

```yaml
api:
  depends_on:
    - postgres  # Waits for container to exist, not to be ready
```

This is unreliable. The API might start before PostgreSQL finishes initializing, causing connection errors.

**Always use `service_healthy`** for databases.

## Reading docker stats — Live Resource Usage

To see real-time resource consumption:

```bash
docker stats
```

Output:

```
CONTAINER     CPU %   MEM USAGE / LIMIT     MEM %    NET I/O
api           0.25%   187.4MiB / 512MiB    36.6%    42.1MB
postgres      2.15%   856.3MiB / 2GiB      41.7%    124MB
redis         0.02%   45.2MiB / 256MiB     17.7%    18MB
```

Watch a specific container:

```bash
docker stats api --no-stream
```

Use `--no-stream` to print once and exit (useful for scripts).

## Hands-On Lab

### Lab 6.1: Observe OOM Kill

#### Step 1: Run a Container with Memory Limit

```bash
docker run -d -m 100M --name memory_test ubuntu:24.04 sleep infinity
```

#### Step 2: Monitor It

```bash
docker stats memory_test
```

You should see it's using ~10MB.

#### Step 3: Allocate Memory Aggressively

```bash
docker exec memory_test bash -c "python -c 'import os; data = bytearray(200 * 1024 * 1024); print(\"Allocated 200MB\")'"
```

You'll see:

```
Killed
```

#### Step 4: Check What Happened

```bash
docker ps | grep memory_test  # Won't be listed (killed)
docker ps -a | grep memory_test  # Will be listed with exit code 137 (OOM)
```

Exit code 137 = 128 + 9 (SIGKILL). The container was OOM-killed.

#### Step 5: Clean Up

```bash
docker rm memory_test
```

### Lab 6.2: Health Checks in Action

#### Step 1: Create a dockerfile with health check

```bash
mkdir -p healthcheck_app
cd healthcheck_app
```

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

RUN pip install fastapi uvicorn

COPY app.py .

HEALTHCHECK --interval=5s --timeout=2s --start-period=3s --retries=2 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app:app", "--host", "0.0.0.0"]
EOF
```

```bash
cat > app.py << 'EOF'
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok"}

@app.get("/")
async def root():
    return {"message": "Hello"}
EOF
```

#### Step 2: Build the Image

```bash
docker build -t healthcheck_app .
```

#### Step 3: Run and Watch Health

```bash
docker run -d -p 8000:8000 --name hc_app healthcheck_app
sleep 2
docker ps | grep hc_app
```

You should see:

```
STATUS: Up 2 seconds (health: starting)
```

Wait a few more seconds:

```bash
sleep 3
docker ps | grep hc_app
STATUS: Up 5 seconds (healthy)
```

#### Step 4: Stop the API and Watch Unhealthy

```bash
docker exec hc_app bash -c "pkill -f uvicorn"
sleep 5
docker ps -a | grep hc_app
```

You should see:

```
STATUS: Exited (137)
```

The container exited because the API crashed and health checks failed.

#### Step 5: Clean Up

```bash
docker rm hc_app
```

### Lab 6.3: Resource Limits in Compose

#### Step 1: Create docker-compose.yml

```yaml
version: '3.9'

services:
  memory_hog:
    image: ubuntu:24.04
    command: sleep infinity
    deploy:
      resources:
        limits:
          memory: 100M
        reservations:
          memory: 50M

  controlled:
    image: ubuntu:24.04
    command: sleep infinity
    deploy:
      resources:
        limits:
          memory: 256M
        reservations:
          memory: 128M
```

#### Step 2: Start and Monitor

```bash
docker compose up -d
docker stats
```

#### Step 3: Allocate Memory in memory_hog

```bash
docker exec memory_hog bash -c "python -c 'data = bytearray(150 * 1024 * 1024); print(\"Allocated\")'"
```

Watch the stats output. The container should be killed (exit code 137) when it tries to exceed 100M.

#### Step 4: Clean Up

```bash
docker compose down
```

## Cheat Sheet: Resource Limits and Health Checks

### Docker Compose Deploy Resource Syntax

```yaml
deploy:
  resources:
    limits:
      cpus: '2'
      memory: 1G
    reservations:
      cpus: '1'
      memory: 512M
```

### Memory Units

- `128m` or `128M` — 128 megabytes
- `1g` or `1G` — 1 gigabyte
- `2048m` — 2 gigabytes

### CPU Units

- `0.5` → 50% of 1 core / 0.5 CPU to be allocated
- `1` → 1 full CPU core
- `2.5` → 2.5 CPU cores

### Health Check Dockerfile Syntax

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD command-to-test || exit 1
```

### Health Check Compose Syntax

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000"]
  interval: 30s
  timeout: 3s
  start_period: 5s
  retries: 3
```

### Viewing Container Health

```bash
docker ps              # Shows (healthy), (unhealthy), or (starting)
docker inspect CONTAINER | grep -A 10 "Health"
```

### Common Health Check Patterns

**HTTP API:**
```dockerfile
CMD curl -f http://localhost:8000/health || exit 1
```

**PostgreSQL:**
```dockerfile
CMD pg_isready -U postgres || exit 1
```

**Redis:**
```dockerfile
CMD redis-cli ping | grep PONG || exit 1
```

**MySQL:**
```dockerfile
CMD mysqladmin ping -h localhost || exit 1
```

## Key Takeaways

- **Set memory limits** to prevent one container from crashing the server
- **Set CPU limits** to prevent one container from starving others
- **Use `depends_on: condition: service_healthy`** to wait for databases to be ready
- **Health checks detect** when services are actually ready, not just started
- **docker stats** shows real-time resource usage
- **start_period** is critical for slow-starting services like databases

Now that you can manage resources and ensure reliability, Module 7 covers deploying to production with zero downtime.
