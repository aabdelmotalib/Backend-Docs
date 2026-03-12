# Module 5: Volumes and Networking

## The Analogy: Files and Addresses

In a city:

- **Volumes** are like storage lockers: secure spaces outside your apartment where you can keep things
- **Networks** are like mailboxes: a communication system that lets neighbors find each other

Without volumes, when you move, you lose everything. Without proper networks, neighbors can't write letters to each other.

**Docker containers are ephemeral**—they disappear when stopped. Volumes and networks solve this.

## Storage Options: Named Volumes vs Bind Mounts vs tmpfs

### Named Volumes (Most Common)

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

A **named volume** is managed by Docker. The actual data lives somewhere on your host (usually `/var/lib/docker/volumes/postgres_data/_data`), but you don't think about the path—Docker handles it.

**Pros:**
- Docker manages the location
- Works on all platforms (Mac, Windows, Linux)
- Easy to back up and migrate
- Can be shared between containers

**Cons:**
- You can't easily browse the files from your laptop
- Requires knowledge of where Docker stores volumes

### Bind Mounts (Development Only)

```yaml
services:
  api:
    volumes:
      - ./src:/app/src  # Host path : Container path
```

A **bind mount** directly mounts a directory from your computer into the container. Changes on your laptop appear instantly in the container.

**Pros:**
- Great for development (edit code on laptop, app reloads)
- Easy to find and edit files
- No Docker abstraction

**Cons:**
- Permissions issues (file ownership differs)
- Works differently on Mac/Windows (slower due to Docker Desktop overhead)
- Security risk (container can modify all your files)
- Not suitable for production

**When to use:**

```yaml
# Development: Hot-reload code
api:
  build: ./api
  volumes:
    - ./api:/app

# Production: NEVER use bind mounts
# Use named volumes or read-only copies
```

### tmpfs (Temporary Storage)

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /cache
```

**tmpfs** creates RAM-backed storage. It's fast but loses data when the container stops.

**Use for:**
- Temporary caches
- Session storage
- Log files
- Anything that doesn't need to persist

**When to use:**

```yaml
worker:
  image: myworker:latest
  tmpfs:
    - /tmp  # Temporary cache directory
```

## When to Use Each Storage Type

| Scenario | Solution |
|----------|----------|
| Database persistence | Named volume |
| File uploads (S3 bucket) | Named volume (temporary) or S3 |
| Development hot-reload | Bind mount |
| Temporary caches | tmpfs |
| Static assets | Bind mount (dev) or COPY in Dockerfile (prod) |
| Logs | Bind mount (dev) or Docker logs (prod) |

## Docker Networking: How Containers Talk

### Bridge Network (Default)

When you run `docker compose up`, Docker creates a **bridge network** connecting all services.

```
┌─────────────────────────────────┐
│  Docker Host (Internal Network) │
├─────────────────────────────────┤
│                                 │
│  ┌──────────┐   ┌──────────┐   │
│  │ API      │   │Database  │   │
│  │172.20.0.2│   │172.20.0.3│   │
│  └──────────┘   └──────────┘   │
│                                 │
│  Bridge: 172.20.0.0/16          │
│  DNS: Internal DNS resolver     │
└─────────────────────────────────┘
```

Each container gets an IP address on the bridge network. Docker's embedded DNS server resolves service names:

- `api:8000` → 172.20.0.2:8000
- `postgres:5432` → 172.20.0.3:5432

Inside the API container, you connect to the database with:

```python
DB_HOST = "postgres"  # Not localhost, not 172.20.0.3
DB_PORT = 5432
```

This is why Docker Compose is so powerful—you reference services by name, not IP addresses.

### Host Network (Advanced)

```yaml
services:
  api:
    network_mode: "host"
```

The container shares the **host's network interface**. No port mapping needed—container ports are host ports directly.

**Use for:** Low-latency applications, debugging network issues. Rarely needed.

### Custom Networks (For Multi-compose-file Architectures)

```yaml
version: '3.9'

services:
  api:
    networks:
      - backend

  postgres:
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

Explicitly define networks for complex setups.

## DNS Resolution Inside Docker Networks

When you're inside an API container and you try to connect to PostgreSQL:

```python
import psycopg2
conn = psycopg2.connect("dbname=myapp user=postgres password=pass host=postgres port=5432")
```

**What happens:**

1. Container's resolver looks up "postgres"
2. Docker's embedded DNS server responds: "postgres = 172.20.0.3"
3. Connection attempt goes to 172.20.0.3:5432
4. Success

**Key insight**: Service names in docker-compose.yml are automatically resolvable. No extra configuration needed.

## The Critical Production Rule: Never Expose Internal Ports

This is the **most important networking rule** in production:

### Wrong (Vulnerable)

```yaml
postgres:
  ports:
    - "5432:5432"  # ❌ Anyone on the network can access

redis:
  ports:
    - "6379:6379"  # ❌ Anyone can read all cached data

minio:
  ports:
    - "9000:9000"  # ❌ Anyone can enumerate buckets

api:
  ports:
    - "8000:8000"  # ✓ OK, this is your API
```

If Postgres is exposed on port 5432, anyone on the network (or the internet if the host is exposed) can:
- Brute-force the password
- Dump the entire database
- Modify all data

### Right (Secure)

```yaml
postgres:
  # No ports: — Not exposed at all
  # Only accessible from other containers via the network

redis:
  # No ports: — Internal only

minio:
  # No ports: — Internal only

nginx:
  ports:
    - "80:80"      # ✓ Only this faces the internet
    - "443:443"

api:
  # No ports: — Only nginx talks to it
```

The **nginx** container is the only thing facing the internet. It forwards requests to the API internally.

### Docker Network vs Host Network

```
┌──────────────────────────────────────┐
│  External Internet                   │
└──────────────┬───────────────────────┘
               │
        ┌──────▼────────────────────────┐
        │  Docker Host Firewall         │
        │  (Port 80 exposed)            │
        └──────┬───────────────────────┘
               │
        ┌──────▼───────────────────────────────┐
        │  Docker Bridge Network               │
        │  172.20.0.0/16 (internal only)      │
        ├─────────────────────────────────────┤
        │                                     │
        │ ┌──────────┐  ┌──────────┐         │
        │ │ nginx    │  │ api      │         │
        │ │Port: 80  │  │Port: 8000│         │
        │ │172.20.0.2│  │172.20.0.3│         │
        │ └──────┬───┘  └──────────┘         │
        │        │                           │
        │ ┌──────▼──────────────────────┐   │
        │ │ postgres                     │   │
        │ │Port: 5432 (internal)         │   │
        │ │172.20.0.4                    │   │
        │ └──────────────────────────────┘   │
        │                                     │
        └─────────────────────────────────────┘
```

External traffic enters only through nginx:80. Everything else is internal.

## Hands-On Lab

### Lab 5.1: Named Volume Persistence

#### Step 1: Create a docker-compose.yml

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: mycompany
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 3

volumes:
  db_data:
```

#### Step 2: Start PostgreSQL

```bash
docker compose up -d
sleep 3
docker compose ps
```

#### Step 3: Create Data

```bash
PGPASSWORD=testpass psql -h localhost -U postgres -d mycompany << 'EOF'
CREATE TABLE employees (id SERIAL PRIMARY KEY, name VARCHAR(100));
INSERT INTO employees (name) VALUES ('Alice'), ('Bob'), ('Charlie');
SELECT * FROM employees;
EOF
```

#### Step 4: Stop and Delete the Container

```bash
docker compose down
docker ps | grep postgres  # Should be empty
```

#### Step 5: Restart Everything

```bash
docker compose up -d
sleep 3
PGPASSWORD=testpass psql -h localhost -U postgres -d mycompany -c "SELECT * FROM employees;"
```

**The data persists!** Even though the container was deleted, the named volume preserved it.

#### Step 6: Clean Up (DELETE DATA)

```bash
docker compose down -v  # -v removes volumes
```

Now the data is gone permanently.

### Lab 5.2: Docker Networks and Service Discovery

#### Step 1: Create a network

```bash
docker network create mynet
```

#### Step 2: Start services on that network

```bash
# Start Redis
docker run -d --name redis --network mynet redis:7-alpine

# Start a Python container that connects to Redis
docker run -it --name python_app --network mynet python:3.12-slim bash
```

#### Step 3: Inside the Python container, test connectivity

```bash
# Inside the container
apt update && apt install -y redis-tools
redis-cli -h redis ping
```

Output: `PONG`

**Key point**: You referenced "redis" by name, not by IP. Docker's DNS resolved it.

#### Step 4: Verify with docker exec

```bash
# From your laptop, not inside the container
docker exec redis redis-cli ping
docker exec python_app redis-cli -h redis ping
```

#### Step 5: Clean up

```bash
docker stop redis python_app
docker rm redis python_app
docker network rm mynet
```

### Lab 5.3: Development Bind Mount

#### Step 1: Create a directory structure

```bash
mkdir -p devapp/src
cd devapp
```

#### Step 2: Create app.py

```bash
cat > src/app.py << 'EOF'
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello v1"}
EOF
```

#### Step 3: Create Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim
WORKDIR /app
RUN pip install fastapi uvicorn
COPY src /app
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--reload"]
EOF
```

#### Step 4: Create docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
version: '3.9'
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app  # Bind mount for hot reload
    command: uvicorn app:app --host 0.0.0.0 --reload
EOF
```

#### Step 5: Start and Test

```bash
docker compose up
```

In another terminal:

```bash
curl http://localhost:8000/
```

#### Step 6: Change the Code (Without Stopping)

Edit `src/app.py`:

```python
@app.get("/")
async def root():
    return {"message": "Hello v2"}
```

Save the file. The container detects the change (uvicorn --reload), restarts the app.

Test:

```bash
curl http://localhost:8000/
```

You'll see `"Hello v2"` without stopping/restarting the container!

#### Step 7: Clean Up

```bash
docker compose down
```

## Cheat Sheet: Volume and Network Commands

### Volume Commands

```bash
docker volume ls                      # List all volumes
docker volume inspect postgres_data   # Details about a volume
docker volume rm postgres_data        # Delete a volume
docker volume prune                   # Delete unused volumes
```

### Network Commands

```bash
docker network ls                     # List networks
docker network inspect mynet          # Details (IP addresses, containers)
docker network create mynet           # Create a custom network
docker network rm mynet              # Delete a network
docker network connect mynet container_name  # Add container to network
docker network disconnect mynet container_name  # Remove from network
```

### Viewing Volumes in Compose

```bash
docker volume ls                      # See all volumes
docker compose down -v                # Remove volumes when stopping
docker volume inspect VOLUME_NAME     # See physical path and metadata
```

## Key Takeaways

- **Named volumes persist data** across container restarts (production)
- **Bind mounts hot-reload code** (development only)
- **tmpfs is RAM-based** (temporary caches)
- **Bridge networks enable service discovery by name** (postgres:5432, redis:6379)
- **Never expose internal ports** (postgres, redis, minio) in production
- **Explicitly expose only the entry point** (nginx, load balancer)
- **Docker embedded DNS** automatically resolves service names on the bridge network

Now that you can persist data and network services, Module 6 covers resource management and health checks to ensure reliability.
