# Module 7: Docker in Production

## The Analogy: Opening a Restaurant

Opening your restaurant (locally on your laptop) is different from running it in Manhattan:

- **Local**: One chef, simple menu, friends as testers
- **Production**: Multiple chefs, complex menu, health inspectors, backup suppliers, 24/7 operation

You can't run production the same way you develop. Different constraints, different standards.

## Dev vs Production Docker Setup

### The Fundamental Difference

**Development (`docker-compose.yml`):**
- Bind mounts for hot code reload
- Exposed ports for debugging
- Dev-friendly logging
- Less strict resource limits
- No redundancy

**Production (`docker-compose.prod.yml`):**
- Code baked into images (no bind mounts)
- Only Nginx exposed on 80/443
- Structured JSON logs
- Strict resource limits
- Restart policies
- Health checks
- Health check before depends_on

## The docker-compose.prod.yml Override Pattern

Create two files:

1. **docker-compose.yml** — Shared config (services, volumes, networks)
2. **docker-compose.prod.yml** — Overrides for production

Then use:

```bash
# Development
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### Example: Bind Mount Override

**docker-compose.yml** (dev):

```yaml
version: '3.9'

services:
  api:
    build: ./api
    volumes:
      - ./api:/app  # Bind mount for hot reload
    environment:
      DEBUG: "true"
    ports:
      - "8000:8000"
```

**docker-compose.prod.yml** (production overrides):

```yaml
version: '3.9'

services:
  api:
    build: ./api  # Same build, but...
    volumes: []   # Remove bind mount (empty list overrides)
    environment:
      DEBUG: "false"
    ports: []     # Remove exposed port (internal only)
    restart: always
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1'
```

When you use both files, the prod file overrides the dev file.

## Never Expose Internal Ports in Production

For a complete production architecture:

**docker-compose.yml:**

```yaml
version: '3.9'

services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"      # Public internet
      - "443:443"    # Public internet

  api:
    build: ./api
    ports:
      - "8000:8000"  # For dev testing

  postgres:
    image: postgres:15
    ports:
      - "5432:5432"  # For dev testing

  redis:
    image: redis:7
    # No ports in dev, but might add for testing

  minio:
    image: minio/minio
    # No ports in dev
```

**docker-compose.prod.yml:**

```yaml
version: '3.9'

services:
  api:
    ports: []      # Remove all port mappings

  postgres:
    ports: []      # Hide from network

  redis:
    ports: []      # Hide from network

  minio:
    ports: []      # Hide from network

  # nginx stays exposed (on 80/443)
```

Now in production:

- Only nginx faces the internet
- All internal services are unreachable from outside
- One hacker can't brute-force your database

## The restart: always Policy

```yaml
services:
  api:
    restart: always

  postgres:
    restart: unless-stopped
```

### restart Policies

| Policy | Behavior |
|--------|----------|
| `no` | Don't restart (default) |
| `always` | Restart if container stops, for any reason |
| `unless-stopped` | Like `always`, but don't restart if manually stopped |
| `on-failure` | Only restart if exit code != 0 |
| `on-failure:5` | Restart max 5 times on failure |

### When to Use Which

- **API/Worker** (stateless): `restart: always`
- **Database** (stateful): `restart: unless-stopped`
- **One-time jobs**: `restart: no` (default)

### How Restart Works

Container crashes:

```
1. Exit (crash)
2. Docker daemon notices
3. Counts as failure
4. If restart: always → Create new container, start
5. Exponential backoff: Wait 100ms, 200ms, 400ms, etc.
```

If a container keeps crashing:
- After 5th restart, Docker waits up to 2 seconds before retrying
- This prevents thrashing (spinning forever)

## Structured Logging in Production

### The Problem with Plain Logs

```
$ docker logs api
INFO:     Uvicorn running on http://0.0.0.0:8000
INFO:     Application startup complete
ERROR: An error occurred
Traceback (most recent call last):
  ...
```

When you have 100 containers, parsing text logs is horrible.

### Solution: JSON-Structured Logging

Make your app output JSON:

```python
import json
import logging

logger = logging.getLogger(__name__)

# Log as JSON
log_entry = {
    "timestamp": "2024-01-15T10:30:45.123Z",
    "level": "INFO",
    "message": "User logged in",
    "user_id": 42,
    "request_id": "req_12345",
    "service": "api"
}
print(json.dumps(log_entry))
```

Now parsing is simple:

```bash
docker logs api | jq '.user_id'  # Extract just user_id
docker logs api | jq 'select(.level == "ERROR")'  # Filter errors
```

### Docker Log Rotation

By default, container logs grow unbounded. Protect your disk:

**docker-compose.yml:**

```yaml
services:
  api:
    image: myapi:latest
    logging:
      driver: "json-file"
      options:
        max-size: "100m"          # Rotate when log hits 100MB
        max-file: "10"            # Keep max 10 old logs (1GB total)
        labels: "service=api"
```

## Updating Containers with Zero Downtime

### The Problem

```bash
docker compose down
# ... All services offline, users see 500 errors ...
docker compose up
# ... Services starting ...
```

This causes downtime. Your users can't use your API.

### Solution: Rolling Updates

Update only the changed service:

```bash
# Pull latest code
git pull

# Rebuild only the API image
docker compose build api

# Start new API, without touching other services
docker compose up -d --no-deps api
```

The `--no-deps` flag means "don't restart services that depend on this one."

### What Happens to Old Requests?

When you update the API:

1. Docker starts a new API container
2. Nginx still has old container in its upstream list (temporarily)
3. Nginx checks health of new API
4. Old API connections keep working until they close
5. New requests route to new container
6. Once all old requests finish, old container stops

This is called **graceful shutdown**.

### Coordinating with Nginx

For true zero downtime, Nginx must know about the new container:

```nginx
# In nginx.conf
upstream api_backend {
    server api:8000;  # Docker DNS resolves to all healthy containers
}

server {
    listen 80;
    location / {
        proxy_pass http://api_backend;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
}
```

When you start a new API container on the same network, Docker's embedded DNS automatically includes it in the `api:8000` resolution.

## The Build-Rebuild-Deploy Pattern

Put this in a **deploy.sh** script:

```bash
#!/bin/bash
set -e  # Exit on any error

echo "Starting deployment..."

# Step 1: Pull latest code
git pull origin main

# Step 2: Rebuild changed images
docker compose build api worker

# Step 3: Start new versions
docker compose up -d --no-deps api worker

# Step 4: Run database migrations (if needed)
docker compose exec -T postgres psql -U appuser -d production << 'EOF'
-- Your migrations here
EOF

# Step 5: Health check
for i in {1..30}; do
    if curl -f http://localhost/health > /dev/null 2>&1; then
        echo "✓ API is healthy"
        exit 0
    fi
    echo "Waiting for API... ($i/30)"
    sleep 1
done

echo "✗ API failed to become healthy"
exit 1
```

## Full Production Example

Here's a realistic production setup:

**docker-compose.yml:**

```yaml
version: '3.9'

services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      api:
        condition: service_healthy
    restart: always
    networks:
      - internal

  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
      SECRET_KEY: ${SECRET_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 30s
    restart: always
    networks:
      - internal
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1'

  worker:
    build: ./worker
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
    depends_on:
      redis:
        condition: service_started
      postgres:
        condition: service_healthy
    restart: on-failure
    networks:
      - internal
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '2'

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped
    networks:
      - internal
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2'
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - internal
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'

networks:
  internal:
    driver: bridge

volumes:
  postgres_data:
```

**docker-compose.prod.yml:**

```yaml
version: '3.9'

services:
  api:
    restart: always
    ports: []  # Remove dev port
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1'
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "10"

  postgres:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 3G
          cpus: '4'

  redis:
    restart: unless-stopped
    ports: []

  worker:
    restart: on-failure
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '4'
```

**deploy.sh:**

```bash
#!/bin/bash
set -e

COMPOSE="docker compose -f docker-compose.yml -f docker-compose.prod.yml"

echo "[1/5] Pulling latest code..."
git pull origin main

echo "[2/5] Building images..."
$COMPOSE build

echo "[3/5] Starting containers..."
$COMPOSE up -d

echo "[4/5] Running migrations..."
$COMPOSE exec -T postgres psql -U appuser -d production << 'EOF'
-- Your migrations
EOF

echo "[5/5] Verifying health..."
for i in {1..30}; do
    if curl -f http://localhost/health > /dev/null 2>&1; then
        echo "✓ System is healthy"
        exit 0
    fi
    echo "Checking... ($i/30)"
    sleep 1
done

echo "✗ System failed health check"
$COMPOSE logs
exit 1
```

Make it executable:

```bash
chmod +x deploy.sh
```

To deploy:

```bash
./deploy.sh
```

## Hands-On Lab

### Lab 7.1: Production Deployment Simulation

#### Step 1: Create docker-compose.yml (dev)

```bash
mkdir -p prod_demo
cd prod_demo
```

```yaml
version: '3.9'

services:
  api:
    build: ./api
    ports:
      - "8000:8000"
    environment:
      VERSION: "v1.0.0"
    restart:no
```

#### Step 2: Create docker-compose.prod.yml (overrides)

```yaml
version: '3.9'

services:
  api:
    restart: always
    ports: []
```

#### Step 3: Create app code

```bash
mkdir -p api
cat > api/Dockerfile << 'EOF'
FROM python:3.12-slim
WORKDIR /app
RUN pip install fastapi uvicorn
COPY app.py .
HEALTHCHECK CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1
CMD ["uvicorn", "app:app", "--host", "0.0.0.0"]
EOF
```

```python
cat > api/app.py << 'EOF'
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
async def root():
    return {"message": "v1.0.0"}

@app.get("/health")
async def health():
    return {"status": "ok"}
EOF
```

#### Step 4: Start Development

```bash
docker compose up -d
curl http://localhost:8000/
docker compose ps
```

#### Step 5: Simulate Production With Overrides

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose ps
# Port 8000 is no longer exposed
```

#### Step 6: Verify You Can't Connect to API Externally

```bash
curl http://localhost:8000/
# Should timeout or fail (can't connect to unexposed port)

# But internally through docker exec works:
docker compose exec api curl http://localhost:8000/health
# Should return {"status": "ok"}
```

#### Step 7: Clean Up

```bash
docker compose down
```

### Lab 7.2: Zero-Downtime Update

#### Step 1: Start with v1.0.0

```bash
docker compose up -d api
sleep 2
```

#### Step 2: Upgrade Code to v2.0.0

Edit `api/app.py`:

```python
return {"message": "v2.0.0"}
```

#### Step 3: Non-Zero-Downtime Update (Old Way)

```bash
# Stop everything
docker compose down

# Start everything
docker compose up -d

# Brief downtime happened!
```

#### Step 4: Zero-Downtime Update (Better Way)

Go back to v1.0.0:

```bash
docker compose up -d api
sleep 2
curl http://localhost:8000/
# Returns v1.0.0
```

Now update to v2.0.0:

```python
return {"message": "v2.0.0"}
```

Deploy:

```bash
# Rebuild only the API
docker compose build api

# Start new API, keep everything else running
docker compose up -d --no-deps api

# Wait for new version to be healthy
sleep 3

# Test
curl http://localhost:8000/
# Returns v2.0.0 with NO downtime!
```

#### Step 5: Clean Up

```bash
docker compose down
```

## Cheat Sheet: Production Docker Best Practices

### Checklist Before Deploying

- [ ] **No dev ports exposed** — Only Nginx/load balancer ports visible
- [ ] **Health checks defined** — All services have HEALTHCHECK
- [ ] **Resource limits set** — Memory and CPU limits for all services
- [ ] **Restart policies configured** — `always` for stateless, `unless-stopped` for stateful
- [ ] **depends_on uses service_healthy** — Not just `service_started`
- [ ] **Secrets in env vars** — Not baked in images or .env files
- [ ] **Structured JSON logging** — Not plain text
- [ ] **Log rotation enabled** — max-size, max-file configured
- [ ] **Volumes are named** — Not bind mounts, data persists
- [ ] **.env in .gitignore** — Secrets don't leak
- [ ] **Override compose file ready** — docker-compose.prod.yml prepared
- [ ] **Deploy script tested** — Deployment is automated

### Deploy Command

```bash
# Build and start
docker compose -f docker-compose.yml -f docker-compose.prod.yml \
  up -d --build

# Update without downtime
docker compose build api
docker compose up -d --no-deps api

# View logs
docker compose logs -f api
```

### Monitoring in Production

```bash
# Real-time stats
docker stats

# View all logs
docker compose logs -f

# Check specific service
docker compose logs api -f --tail 100

# Structured query (if using JSON logs)
docker logs container_id | jq '.level' | sort | uniq -c
```

## Key Takeaways

- **Production and dev are different** — Use docker-compose.prod.yml overrides
- **Never expose internal services** — Only Nginx on 80/443
- **Restart policies prevent cascading failures** — `always` and `unless-stopped`
- **Health checks enable zero-downtime updates** — Don't start services before dependencies are healthy
- **JSON logging makes debugging easier** — Structure your logs
- **--no-deps allows isolated updates** — Update one service without restarting others
- **Automate deployment** — deploy.sh ensures consistency

You've now mastered Docker from concept to production. The Linux section teaches the underlying OS fundamentals that make Docker possible.
