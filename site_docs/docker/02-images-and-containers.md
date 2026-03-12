# Module 2: Images and Containers

## The Analogy: The Recipe vs The Cake

A Docker **image** is like a recipe:

- Read-only instructions
- Describes every ingredient and step
- Exact measurements
- Reproducible

A Docker **container** is like a baked cake:

- An instance of the recipe
- You can have multiple cakes from one recipe
- You can eat a cake (run it), freeze it (stop it), or throw it away (delete it)
- Changes to one cake don't affect others

The recipe (image) never changes. But each cake you bake is slightly different—different oven, different timing. Similarly, each container (instance of the image) has its own filesystem, its own running processes, and its own network interfaces.

## What is a Docker Image?

A Docker image is a **layered filesystem snapshot**. Think of it as a stack of pancakes:

```
┌─────────────────────────┐
│  Layer 4: Your code     │  (what you COPY in)
├─────────────────────────┤
│  Layer 3: Dependencies  │  (what you RUN pip install)
├─────────────────────────┤
│  Layer 2: Python 3.12   │  (what apt install gives you)
├─────────────────────────┤
│  Layer 1: Ubuntu 24.04  │  (the FROM layer)
└─────────────────────────┘
```

Each layer is **immutable** (unchangeable). When you build an image, Docker optimizes by caching layers. If you rebuild an image and only change Layer 4, Docker skips rebuilding Layers 1-3.

### Image Anatomy

Every image has:

- **Base image** — The foundation (Ubuntu, Alpine, Python, etc.)
- **Metadata** — Environment variables, exposed ports, working directory
- **Filesystem** — Files and directories bundled into the image
- **Entry point** — The command that runs when you start a container
- **ID** — A SHA256 hash that uniquely identifies the image

### Image Size Matters

```bash
docker images
```

Output:
```
REPOSITORY      TAG       IMAGE ID      CREATED       SIZE
ubuntu          24.04     b6548eacb063  2 weeks ago   77.9MB
python          3.12-slim fffc5dd71908  3 days ago    179MB
postgres        15        31e3ecd97f0a  1 week ago    371MB
myapp           latest    a2c8d9e1f5b3  1 hour ago    243MB
```

- **Ubuntu 24.04 (77.9MB)**: Just the OS, minimal
- **Python 3.12-slim (179MB)**: Python runtime + minimal dependencies
- **PostgreSQL 15 (371MB)**: Full database engine
- **Your app (243MB)**: Probably: Python slim (179MB) + your dependencies (64MB)

**Important**: Images are always compressed and deduplicated. If you have 5 containers running `python:3.12-slim`, you have only one copy of that 179MB image on disk.

## Image Registries: Where Images Live

### Docker Hub (docker.io)

The most popular registry. Public images anyone can use:

```bash
# These commands look for images on Docker Hub
docker pull ubuntu:24.04
docker pull python:3.12-slim
docker pull postgres:15
docker pull nginx:latest
```

When you specify just the image name, Docker assumes `docker.io/library/IMAGE`, which is the official repository.

### GitHub Container Registry (ghcr.io)

Used for private images and open-source projects:

```bash
docker pull ghcr.io/myorg/myapp:v1.2.3
```

### Private Registries

Large companies run their own registries (AWS ECR, Azure Container Registry) for proprietary images.

!!! warning
    Never put secrets in images. Use environment variables and secrets management instead. A secret baked into an image is a liability.

## How Image Layers Work (Critical for Speed)

This is where Docker gets powerful. Let's say you build an image three times:

**First build** (from scratch):
```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3 python3-pip
RUN pip install fastapi uvicorn
COPY app.py .
RUN chmod +x app.py
```

Docker builds 5 layers:
1. `FROM ubuntu:24.04` → 77.9MB (cached)
2. `RUN apt update...` → 280MB (cached)
3. `RUN pip install...` → 45MB (cached)
4. `COPY app.py .` → +2KB (cached)
5. `RUN chmod...` → +0KB (cached)

**Second build** (only app.py changed):
```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3 python3-pip
RUN pip install fastapi uvicorn
COPY app.py .  # NEW CONTENT
RUN chmod +x app.py
```

Docker reuses layers 1-3 **from cache**, rebuilds only layers 4-5. Takes 0.5 seconds instead of 45 seconds.

**Third build** (added a new pip dependency):
```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3 python3-pip python3-psycopg2  # CHANGED
RUN pip install fastapi uvicorn sqlalchemy  # CHANGED
COPY app.py .
RUN chmod +x app.py
```

Layer 1 is cached, layer 2 is invalidated (apt command changed), so Docker rebuilds layers 2-5.

### Layer Caching Rules

**A layer is reused if and only if:**
1. The instruction is identical to the previous build
2. All layers before it are identical and reused

**Warning**: Order matters! Put things that change rarely at the top, things that change often at the bottom:

```dockerfile
FROM ubuntu:24.04  # Rarely changes
RUN apt update...  # Rarely changes
RUN pip install...  # Semi-often changes (when deps update)
COPY . .           # Often changes (when you code)
RUN python setup.py # Every build (consequence of COPY change)
```

This is why Docker images build fast after the first time.

## Working with Images and Containers

### docker pull — Download an Image

```bash
docker pull python:3.12-slim
docker pull postgres:15-alpine
docker pull redis:7.4
```

The tag (`:3.12-slim`, `:15-alpine`) specifies the version. If you omit it, Docker uses `:latest`, which is often not what you want in production.

!!! danger
    Never use `:latest` in production. It will pull different versions on different servers. Always use explicit version tags like `:3.12`, `:15`, `:7.4`.

### docker images — List Your Images

```bash
docker images
```

Output:
```
REPOSITORY    TAG           IMAGE ID      CREATED       SIZE
python        3.12-slim     19fe108f6670  2 days ago    177MB
postgres      15-alpine     e0b43f97d94d  1 week ago    208MB
redis         7.4           c08999f0d12d  3 days ago    139MB
```

### docker run — Create and Start a Container

Already covered in Module 1, but here's the full command:

```bash
docker run [OPTIONS] IMAGE [COMMAND]
```

**Most common options**:

```bash
docker run -d --name myapi -p 8000:8000 python:3.12-slim python app.py
```

| Flag | Means |
|------|-------|
| `-d` | Detach—run in background |
| `--name myapi` | Give the container a name |
| `-p 8000:8000` | Expose port 8000 (host:container) |
| `python:3.12-slim` | The image to use |
| `python app.py` | The command to run inside |

### docker ps — List Containers

```bash
docker ps           # Running containers only
docker ps -a        # All containers (running + stopped)
docker ps -n 5      # Last 5 containers
```

Output:
```
CONTAINER ID  IMAGE      NAMES     PORTS                STATUS
abc123def     python... myapi     0.0.0.0:8000->8000  Up 2 minutes
```

### docker stop / kill / rm

```bash
docker stop myapi      # Graceful shutdown (SIGTERM)
docker kill myapi      # Force shutdown (SIGKILL)
docker rm myapi        # Delete the container completely
docker rm -f myapi     # Force delete (even if running)
```

**Key difference**:
- `stop`: Gives the app 10 seconds to shut down gracefully
- `kill`: Immediate termination, app doesn't get to clean up
- Use `stop` for databases and stateful apps, use `kill` only when necessary

### docker logs — View Output

```bash
docker logs myapi          # Print all logs
docker logs myapi -f       # Follow logs (like tail -f)
docker logs myapi --tail 50  # Last 50 lines
```

### docker exec — Run a Command Inside a Container

```bash
# Start a PostgreSQL container
docker run -d --name postgres_db postgres:15

# Connect with psql from outside the container
docker exec -it postgres_db psql -U postgres
```

The `-it` flags mean "interactive + allocate a TTY" so you can type commands.

## Real-World Example: PostgreSQL Container

Let's run a real database and connect to it:

```bash
# Start PostgreSQL
docker run -d \
  --name postgres_db \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:15
```

Flags explained:
- `-e POSTGRES_PASSWORD=mypassword`: Set password via environment variable
- `-e POSTGRES_DB=myapp`: Create a database named 'myapp'
- `-p 5432:5432`: Expose port 5432 so you can connect from your laptop

Now connect from your laptop (not from inside the container):

```bash
# Install the PostgreSQL client (if not already installed)
sudo apt install postgresql-client

# Connect
psql -h localhost -U postgres -d myapp
```

You'll be prompted for a password: `mypassword`

Once connected:

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100)
);

INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
SELECT * FROM users;
```

Now stop and delete the container:

```bash
docker stop postgres_db
docker rm postgres_db
```

**Key insight**: The database data is gone. In the next module, we'll use **volumes** to persist data.

## Image Tagging and Versioning

When you build an image, you should tag it:

```bash
docker build -t myapp:1.0.0 .
docker build -t myapp:latest .
docker build -t ghcr.io/myorg/myapp:1.0.0 myapp:latest
```

**Convention**:
- `myapp:1.0.0` — Specific version (for production)
- `myapp:latest` — Latest version (for development)
- `ghcr.io/myorg/myapp:1.0.0` — Registry/org/app:tag format

## Hands-On Lab

### Lab 2.1: Pull and Run PostgreSQL

**Goal**: Run a database container and verify you can connect to it.

#### Step 1: Pull PostgreSQL Image

```bash
docker pull postgres:15
docker images | grep postgres
```

You should see `postgres:15` with ~371MB.

#### Step 2: Run the Container

```bash
docker run -d \
  --name mydb \
  -e POSTGRES_PASSWORD=securepass123 \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_DB=production \
  -p 5432:5432 \
  postgres:15
```

#### Step 3: Wait for It to Be Ready

```bash
# Check if it's running
docker ps | grep mydb

# View the logs
docker logs mydb
```

You should see: `database system is ready to accept connections`

#### Step 4: Connect with psql

```bash
# Install psql client if needed
sudo apt install -y postgresql-client

# Connect to the database
PGPASSWORD=securepass123 psql -h localhost -U appuser -d production
```

#### Step 5: Create a Table and Insert Data

```sql
-- Inside psql
CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  title VARCHAR(200),
  content TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO articles (title, content) VALUES ('Docker Guide', 'This is comprehensive');
SELECT * FROM articles;

-- Exit
\q
```

#### Step 6: Verify from Another Connection

```bash
# Connect again (you can open another terminal)
PGPASSWORD=securepass123 psql -h localhost -U appuser -d production -c "SELECT * FROM articles;"
```

#### Step 7: Clean Up

```bash
docker stop mydb
docker rm mydb
```

**Key insight**: The data was deleted when you deleted the container. This is normal for containers—they're ephemeral.

### Lab 2.2: Understand Image Layers

```bash
# Pull an image
docker pull nginx:latest

# Inspect the image (see layers)
docker image inspect nginx:latest | grep -A 20 "Layers"
```

You'll see a list of layer IDs. Each one is a snapshot of the filesystem at that point in the build.

### Lab 2.3: Tag an Image

```bash
# Pull an official image
docker pull ubuntu:24.04

# Create a tag (alias)
docker tag ubuntu:24.04 myos:v1

# List images
docker images | grep -E "ubuntu|myos"
```

Both rows point to the same image (same IMAGE ID). Tags are just labels.

## Cheat Sheet: Full docker run Flag Reference

| Flag | Example | Meaning |
|------|---------|---------|
| `--name` | `--name myapi` | Give the container a name |
| `-d` | `-d` | Run in background (detached) |
| `-it` | `-it` | Interactive + TTY (for shells) |
| `-p` | `-p 8000:8000` | Port mapping (host:container) |
| `-e` | `-e DEBUG=true` | Environment variable |
| `-v` | `-v data:/data` | Mount a volume |
| `--rm` | `--rm` | Auto-delete when stopped |
| `--restart` | `--restart always` | Restart policy |
| `--network` | `--network mynet` | Connect to a network |
| `-u` | `-u appuser` | Run as a user |
| `-w` | `-w /app` | Working directory |
| `-c` | `-c 512` | CPU shares (relative) |
| `-m` | `-m 512m` | Memory limit (e.g., 256m, 1g) |
| `--cpus` | `--cpus 1.5` | Max CPUs (e.g., 0.5, 2) |
| `--expose` | `--expose 8000` | Document port (metadata only) |

## Key Takeaways

- **Images are immutable blueprints**, containers are running instances
- **Layers enable caching**, making rebuilds fast
- **Tags are version labels**, not different images
- **Registries (Docker Hub, etc.) distribute images** across the internet
- **Port mapping (-p)** connects container ports to the host
- **Environment variables (-e)** configure containers without changing code
- **Volumes (-v)** persist data—covered in Module 5

Now that you understand images and containers, Module 3 teaches you to build your own images with Dockerfiles.
