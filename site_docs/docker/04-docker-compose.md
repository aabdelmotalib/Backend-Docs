# Module 4: Docker Compose

## The Analogy: The Orchestra

An **orchestra** has many instruments:

- Violins need music paper and a stand
- Drums need space and sound insulation
- Everyone needs to hear the conductor at exactly the right tempo

Coordinating them manually would be chaos. Instead, the **conductor** (Docker Compose) ensures every instrument:
- Starts in the right order
- Gets the resources it needs
- Communicates with others at the right time
- Stops gracefully when the piece ends

**Docker Compose** is the conductor for your microservices.

## What Docker Compose Does (And Why You Need It)

Running a 3-service system without Docker Compose:

```bash
# Terminal 1: Start PostgreSQL
docker run -d --name postgres -e POSTGRES_PASSWORD=pass postgres:15

# Terminal 2: Start Redis
docker run -d --name redis redis:7

# Terminal 3: Start your API
docker run -d --name api --link postgres --link redis \
  -e DB_HOST=postgres -e REDIS_HOST=redis \
  myapi:latest

# Now manage 3 separate containers manually, handle restarts, networking, etc.
```

With Docker Compose:

```bash
docker compose up
# All 3 services start in order, with networking configured, in one command.
```

**Docker Compose solves:**

- Starting services in dependency order
- Creating a shared network so services find each other by name
- Setting environment variables per service
- Mounting volumes for data persistence
- Resource limits per service
- Health checks and restart policies
- Everything in a single YAML file that version-controls

## The docker-compose.yml Structure

A basic example:

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secretpassword
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 3s
      retries: 3

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  api:
    build: ./api
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    environment:
      DATABASE_URL: postgresql://postgres:secretpassword@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    ports:
      - "8000:8000"
    restart: always

volumes:
  postgres_data:
```

Now let's explain every line.

## Service Fields Explained

### `version`

```yaml
version: '3.9'
```

Specifies the Compose file format. `3.9` is the latest stable version. Don't use `2.1` (deprecated) or experimental versions.

### `services`

The list of containers to run. Each service becomes a container.

### `image` — Use a Pre-Built Image

```yaml
postgres:
  image: postgres:15
```

Uses Docker Hub's official PostgreSQL image, version 15.

### `build` — Build from a Dockerfile

```yaml
api:
  build: ./api
```

Builds the image from `./api/Dockerfile`. Docker Compose re-builds if the Dockerfile or code changes.

Advanced:

```yaml
api:
  build:
    context: ./api
    dockerfile: Dockerfile
    args:
      BUILD_ENV: production
```

### `environment`

```yaml
postgres:
  environment:
    POSTGRES_PASSWORD: secretpassword
    POSTGRES_DB: myapp
    POSTGRES_USER: postgres
```

Sets environment variables inside the container.

Better: Use a `.env` file:

```yaml
postgres:
  environment:
    POSTGRES_PASSWORD: ${DB_PASSWORD}
    POSTGRES_DB: ${DB_NAME}
```

Create `.env`:

```
DB_PASSWORD=secretpassword
DB_NAME=myapp
```

Docker Compose automatically loads `.env` variables.

!!! danger
    Don't commit `.env` with secrets to version control. Add `.env` to `.gitignore`. In production, use a secrets management system.

### `ports` — Expose Ports

```yaml
postgres:
  ports:
    - "5432:5432"  # host:container

api:
  ports:
    - "8000:8000"
```

Makes port 5432 on your computer map to port 5432 inside the PostgreSQL container.

### `volumes` — Persistent Storage

```yaml
postgres:
  volumes:
    - postgres_data:/var/lib/postgresql/data

# At the end of the file:
volumes:
  postgres_data:
```

Creates a named volume `postgres_data` and mounts it to `/var/lib/postgresql/data` inside the container. Data persists even when the container stops.

### `healthcheck` — Is the Service Healthy?

```yaml
postgres:
  healthcheck:
    test: ["CMD", "pg_isready", "-U", "postgres"]
    interval: 10s
    timeout: 3s
    retries: 3
    start_period: 40s
```

Every 10 seconds, Docker runs `pg_isready`. If it returns 0 (success), the container is healthy. After 3 failed checks, the container is marked unhealthy.

- `interval: 10s` — Check every 10 seconds
- `timeout: 3s` — Wait up to 3 seconds for the check
- `retries: 3` — Mark unhealthy after 3 failures
- `start_period: 40s` — Give the service 40 seconds to start before checking

### `depends_on` — Start Order and Prerequisites

#### Old Way (Service Started)

```yaml
api:
  depends_on:
    - postgres
    - redis
```

Waits for `postgres` and `redis` containers to **exist and start**, but not necessarily be healthy. This is unreliable.

#### Better Way (Service Healthy)

```yaml
api:
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_started
```

`service_healthy`: Wait until the container's healthcheck passes.
`service_started`: Wait until the container starts (no health check).

**Critical**: Always use `service_healthy` for databases. If the API starts before PostgreSQL is ready, it crashes.

### `restart`

```yaml
api:
  restart: always

worker:
  restart: on-failure

postgres:
  restart: no
```

Options:

- `no` — Don't restart (default)
- `always` — Restart if the container stops, even if it exited successfully
- `on-failure` — Restart only if the container exits with a non-zero code
- `unless-stopped` — Like `always`, but don't restart if the container was manually stopped

**Strategy**: Use `always` for stateless services (API, worker), `unless-stopped` for stateful services (database).

### `environment` vs `.env` vs `--env-file`

```yaml
# Method 1: Inline (development only)
api:
  environment:
    DEBUG: "true"
    DATABASE_URL: postgresql://...

# Method 2: From .env file (better)
api:
  environment:
    DEBUG: ${DEBUG}
    DATABASE_URL: ${DATABASE_URL}
```

In `.env`:

```
DEBUG=true
DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
```

Docker Compose loads `.env` automatically.

## Docker Compose Commands

### docker compose up — Start All Services

```bash
docker compose up             # Start, show logs
docker compose up -d          # Start in background
docker compose up --build     # Rebuild images first
docker compose up -d --build  # Build and start in background
```

### docker compose down — Stop All Services

```bash
docker compose down           # Stop and remove containers
docker compose down -v        # Also remove volumes (!)
```

!!! danger
    `docker compose down -v` deletes volumes, destroying all data. Use with caution.

### docker compose logs — View Output

```bash
docker compose logs             # All service logs
docker compose logs -f          # Follow logs (like tail -f)
docker compose logs postgres    # Just PostgreSQL logs
docker compose logs --tail 50   # Last 50 lines
```

### docker compose ps — List Services

```bash
docker compose ps
```

Output:

```
NAME          STATUS         PORTS
postgres      Up (healthy)   5432/tcp
redis         Up             6379/tcp
api           Up             8000/tcp
```

### docker compose exec — Run Commands Inside a Service

```bash
docker compose exec postgres psql -U postgres -d myapp
docker compose exec api python manage.py migrate
docker compose exec -it api bash
```

### docker compose restart — Restart Services

```bash
docker compose restart           # All services
docker compose restart api       # Just API
docker compose restart api postgres  # Multiple
```

### docker compose pull — Update Images

```bash
docker compose pull  # Download latest versions of images
docker compose up -d  # Start with new images
```

## Real-World Example: The PDF SaaS Platform

Here's the actual `docker-compose.yml` for a production PDF processing platform:

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
      DATABASE_URL: postgresql://appuser:${DB_PASSWORD}@postgres:5432/production
      REDIS_URL: redis://redis:6379
      MINIO_URL: http://minio:9000
      MINIO_ACCESS_KEY: ${MINIO_KEY}
      MINIO_SECRET_KEY: ${MINIO_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
      minio:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: always
    networks:
      - internal

  worker:
    build: ./worker
    environment:
      DATABASE_URL: postgresql://appuser:${DB_PASSWORD}@postgres:5432/production
      REDIS_URL: redis://redis:6379
      MINIO_URL: http://minio:9000
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
      POSTGRES_DB: production
      POSTGRES_USER: appuser
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 3s
      retries: 5
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - internal

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    networks:
      - internal

  minio:
    image: minio/minio:latest
    environment:
      MINIO_ROOT_USER: ${MINIO_KEY}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET}
    command: server /minio_data
    volumes:
      - minio_data:/minio_data
    restart: always
    networks:
      - internal

  clamav:
    image: clamav/clamav:latest
    restart: unless-stopped
    networks:
      - internal

  flower:
    image: mher/flower:latest
    environment:
      CELERY_BROKER_URL: redis://redis:6379/0
      CELERY_RESULT_BACKEND: redis://redis:6379/0
    ports:
      - "5555:5555"
    depends_on:
      - redis
    restart: on-failure
    networks:
      - internal

networks:
  internal:
    driver: bridge

volumes:
  postgres_data:
  minio_data:
```

**What's happening:**

- **nginx**: Reverse proxy facing the internet (ports 80/443 exposed)
- **api**: FastAPI backend, only internal network, health check required
- **worker**: Celery worker, resource-limited (1GB RAM, 2 CPUs)
- **postgres**: Database, persistent volume, requires health check
- **redis**: Cache and message broker
- **minio**: S3-compatible file storage, persistent volume
- **clamav**: Antivirus scanner
- **flower**: Celery monitoring UI
- **networks**: All services on `internal` network for secure communication
- **depends_on**: Wait for dependencies to be healthy before starting

## Hands-On Lab

### Lab 4.1: Basic Docker Compose with 3 Services

#### Step 1: Create Project Structure

```bash
mkdir -p lab_compose
cd lab_compose
```

#### Step 2: Create docker-compose.yml

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: testdb
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 3
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

  api:
    image: python:3.12-slim
    command: python -m http.server
    working_dir: /app
    ports:
      - "8000:8000"
    environment:
      DB_HOST: postgres
      REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: always

volumes:
  postgres_data:
```

#### Step 3: Start All Services

```bash
docker compose up -d
```

#### Step 4: Check Status

```bash
docker compose ps
```

You should see all 3 services running and healthy.

#### Step 5: Test Connectivity

Connect to PostgreSQL:

```bash
PGPASSWORD=mypassword psql -h localhost -U postgres -d testdb -c "SELECT version();"
```

Test Redis:

```bash
redis-cli -h localhost ping
```

Call the API:

```bash
curl localhost:8000 | head -20
```

#### Step 6: View Logs

```bash
docker compose logs -f postgres
docker compose logs -f redis
docker compose logs -f api
```

Press Ctrl+C to stop following logs.

#### Step 7: Stop Everything

```bash
docker compose down
```

### Lab 4.2: Real FastAPI + PostgreSQL

#### Create Dockerfile (api/Dockerfile)

```bash
mkdir -p api
cat > api/Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

RUN useradd -m appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser main.py .

USER appuser

HEALTHCHECK --interval=5s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
EOF
```

#### Create requirements.txt

```bash
cat > api/requirements.txt << 'EOF'
fastapi==0.104.1
uvicorn==0.24.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
EOF
```

#### Create main.py

```bash
cat > api/main.py << 'EOF'
from fastapi import FastAPI
import os
from sqlalchemy import create_engine, text

app = FastAPI()

db_url = os.getenv("DATABASE_URL", "postgresql://postgres:password@postgres:5432/testdb")
engine = create_engine(db_url)

@app.get("/health")
async def health():
    try:
        with engine.connect() as conn:
            conn.execute(text("SELECT 1"))
        return {"status": "ok", "database": "connected"}
    except:
        return {"status": "error", "database": "disconnected"}

@app.get("/")
async def root():
    return {"message": "API is running"}
EOF
```

#### Create docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 3
    volumes:
      - postgres_data:/var/lib/postgresql/data

  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql://postgres:mypassword@postgres:5432/testdb
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
    restart: always

volumes:
  postgres_data:
EOF
```

#### Start and Test

```bash
docker compose up -d
sleep 5
curl http://localhost:8000/health
curl http://localhost:8000/
docker compose logs -f api
docker compose down
```

## Cheat Sheet: Docker Compose Command Reference

| Command | Use |
|---------|-----|
| `docker compose up` | Start all services in foreground |
| `docker compose up -d` | Start in background |
| `docker compose up --build` | Rebuild and start |
| `docker compose down` | Stop and remove containers |
| `docker compose down -v` | Stop and remove volumes (**WARNING**) |
| `docker compose ps` | List services and status |
| `docker compose logs SERVICE` | View logs for a service |
| `docker compose logs -f` | Follow logs |
| `docker compose exec SERVICE bash` | Run command in a service |
| `docker compose restart SERVICE` | Restart a service |
| `docker compose stop` | Pause components (keep containers) |
| `docker compose start` | Resume paused services |
| `docker compose pull` | Download latest images |
| `docker compose config` | Validate compose file |

### Troubleshooting Common Issues

**Services fail to start with "connection refused"**

→ Use `condition: service_healthy` in `depends_on`, not just `service_started`.

**Port already in use**

→ Change the port mapping: `- "9000:8000"` instead of `- "8000:8000"`

**Volume not persisting**

→ Verify the volume mount path inside the container matches the data directory.

**Containers can't reach each other by name**

→ All services must be on the same network (Compose creates one automatically, but verify with `docker network ls`).

Now that you can orchestrate multiple services, Module 5 covers the infrastructure layer: volumes and networking.
