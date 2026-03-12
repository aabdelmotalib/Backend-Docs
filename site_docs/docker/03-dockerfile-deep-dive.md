# Module 3: Dockerfile Deep Dive

## The Analogy: The Cookbook

A **Dockerfile** is like a detailed cookbook:

- Precise ingredients (base image, dependencies)
- Step-by-step instructions (apt install, pip install, copy files)
- What to run when someone uses your recipe (ENTRYPOINT/CMD)
- Comments explaining the reasoning

When you run `docker build`, Docker executes every step in order, building layers, caching where possible, and finally producing an image—your finalized dish.

A bad Dockerfile is like a bad recipe: it works, but it wastes ingredients, creates huge files, or takes forever to execute.

## Every Dockerfile Instruction Explained

### FROM — The Base Image

**Every Dockerfile must start with FROM.**

```dockerfile
FROM ubuntu:24.04
FROM python:3.12-slim
FROM node:20-alpine
FROM scratch  # Empty starting point (advanced)
```

This chooses your base layer—the foundation image that contains the OS and maybe some pre-installed tools.

**When to use which base:**

| Base | Size | Use Case |
|------|------|----------|
| `ubuntu:24.04` | 77MB | Need flexibility, don't mind size |
| `python:3.12-slim` | 177MB | Python apps, minimal. **Recommended** |
| `python:3.12-alpine` | 51MB | Python apps, smaller. Less compatible |
| `node:20-slim` | 239MB | Node.js apps |
| `golang:1.20` | 820MB | Go apps (large but stable) |
| `scratch` | 0MB | Compiled binaries only (advanced) |

!!! tip
    Use `-slim` variants for most apps. They're smaller than full images and include essential libraries. `-alpine` is smaller but sometimes has library compatibility issues with Python.

### WORKDIR — Set Working Directory

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
```

`WORKDIR` is like `cd` for the container. All subsequent commands run in this directory.

**Best practice**: Always set WORKDIR, even if it's `/app`. Don't put stuff in `/` or `/home`.

### COPY — Copy Files from Your Computer

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt requirements.txt  # Copy host file to container
COPY . .  # Copy everything in current dir to /app in container
RUN pip install -r requirements.txt
COPY api.py api.py  # Copy after pip install (takes advantage of caching)
```

`COPY HOST_PATH CONTAINER_PATH`

**Key insight**: Docker respects each instruction as a separate layer. If you `COPY . .` before `RUN pip install`, then change your code, Docker has to rebuild pip install (slow). If you `COPY requirements.txt`, then `RUN pip install`, then `COPY . .`, Docker uses the pip-installed layer even when code changes.

### RUN — Execute a Command

```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3 python3-pip
RUN pip install requests

FROM python:3.12-slim
RUN pip install requests
```

`RUN` executes a shell command and creates a layer with the results.

**Optimization tip**: Chain commands with `&&` to reduce layers:

Bad:
```dockerfile
RUN apt update
RUN apt install -y curl
RUN apt install -y wget
# 3 layers, 100MB combined
```

Good:
```dockerfile
RUN apt update && apt install -y curl wget
# 1 layer, 40MB
```

Each layer adds size. Minimize layers by chaining commands.

### ENV — Set Environment Variables

```dockerfile
FROM python:3.12-slim
ENV PYTHONUNBUFFERED=1  # Python logs immediately, not buffered
ENV PORT=8000
ENV DATABASE_URL=postgres://localhost/mydb
```

Environment variables are baked into the image. They're available inside containers but can be overridden at runtime:

```bash
docker run -e PORT=9000 myapp:latest
```

### EXPOSE — Document Ports

```dockerfile
FROM python:3.12-slim
EXPOSE 8000  # This is metadata only!
```

`EXPOSE` is **documentation**. It doesn't actually expose the port. When someone reads the Dockerfile, they know the app listens on 8000.

To actually expose the port when running:

```bash
docker run -p 8000:8000 myapp:latest
```

### CMD vs ENTRYPOINT — What Runs When the Container Starts

This is where most people get confused.

#### CMD — Default Command

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

When you run the container without specifying a command:

```bash
docker run myapp:latest
# Runs: python app.py
```

You can override CMD:

```bash
docker run myapp:latest python other.py
# Runs: python other.py (ignores CMD)
```

#### ENTRYPOINT — The Fixed Entry Point

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
ENTRYPOINT ["python", "app.py"]
```

When you run the container:

```bash
docker run myapp:latest
# Runs: python app.py

docker run myapp:latest --debug
# Runs: python app.py --debug (appends arguments)
```

With ENTRYPOINT, arguments are appended. With CMD, arguments replace the command.

#### When to Use Which?

use `ENTRYPOINT` if your container has one main purpose (the app always runs):

```dockerfile
# FastAPI API server
ENTRYPOINT ["uvicorn", "api:app"]
```

Use `CMD` if you want flexibility (maybe run a shell, maybe run the app):

```dockerfile
# Celery worker (might want to inspect or debug)
CMD ["celery", "-A", "tasks", "worker"]
```

**Common pattern**: Use both:

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
# docker run myapp:latest → python app.py --port 8000
# docker run myapp:latest --port 9000 → python app.py --port 9000
```

## Multi-Stage Builds — Optimize Image Size

A problem: your code is 50MB, your build dependencies are 500MB. You don't need the build tools in production.

**Solution: Multi-stage builds.**

```dockerfile
# Stage 1: Builder (compile/prepare)
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user -r requirements.txt  # Install to /root/.local

# Stage 2: Runtime (final image)
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local  # Copy only installed packages
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

What happens:

1. Stage 1 builds and installs everything (big intermediate image)
2. Stage 2 only copies the compiled/installed files (lean final image)
3. The final image doesn't include the compiler, build tools, or source code

Image size difference:
- Without multi-stage: 600MB (Python + deps + build tools)
- With multi-stage: 250MB (Python + deps only)

!!! tip
    Multi-stage builds are essential for production images. Always use them.

## Build Context and .dockerignore

When you run `docker build`, Docker sends your entire directory to the daemon. This is the **build context**.

```bash
docker build -t myapp .
# Sends everything in the current directory to the daemon
```

If you have large files (node_modules, __pycache__, venv), they get sent even though you don't need them.

**Solution: .dockerignore**

Create `.dockerignore` in your project root:

```
__pycache__
*.pyc
venv/
.env
.git
node_modules/
build/
dist/
.pytest_cache/
*.egg-info/
```

Now `docker build` skips these. Builds are much faster.

!!! warning
    Don't add secrets files to .dockerignore hoping they won't get into the image. If they're in COPY . ., they'll still be copied. Use secrets management instead (covered in Module 7).

## Real-World Example: FastAPI App Dockerfile

Here's a production-ready Dockerfile for a FastAPI application:

```dockerfile
# Multi-stage: Stage 1 - Builder
FROM python:3.12-slim AS builder

WORKDIR /build

# Copy requirements first (layer caching)
COPY requirements.txt .

# Install dependencies to a local dir
RUN pip install --user --no-cache-dir -r requirements.txt

# Multi-stage: Stage 2 - Runtime
FROM python:3.12-slim

WORKDIR /app

# Create a non-root user for security
RUN useradd -m appuser

# Copy installed packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser api/ ./api/
COPY --chown=appuser:appuser config.py main.py ./

# Set Python path
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# Expose port
EXPOSE 8000

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run the application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Let's break down every line:

| Line | Why |
|------|-----|
| `FROM python:3.12-slim AS builder` | Start builder stage, use slim base |
| `COPY requirements.txt .` | Copy deps file before full directory |
| `RUN pip install --user --no-cache-dir` | Use --user to avoid root install, --no-cache-dir to save space |
| `FROM python:3.12-slim` | Start fresh for runtime stage |
| `RUN useradd -m appuser` | Create non-root user (security) |
| `COPY --from=builder /root/.local...` | Copy only the installed packages from builder |
| `COPY --chown=appuser:appuser...` | Copy code and own it with appuser |
| `ENV PYTHONUNBUFFERED=1` | Python logs immediately |
| `USER appuser` | Switch to non-root user before running |
| `HEALTHCHECK` | Kubernetes/orchestrators can check if app is healthy |
| `CMD ["uvicorn", "main:app"...` | Start the app |

### requirements.txt

```
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
python-jose==3.3.0
```

### main.py

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/health")
async def health():
    return JSONResponse({"status": "ok"})

@app.get("/")
async def root():
    return {"message": "Hello World"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Celery Worker Example

For a background worker, the Dockerfile is simpler:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN useradd -m worker

COPY --chown=worker:worker requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=worker:worker tasks/ ./tasks/
COPY --chown=worker:worker worker.py .

USER worker

CMD ["celery", "-A", "tasks", "worker", "--loglevel=info"]
```

Workers don't expose ports (they listen to message queues), don't need health checks, and don't serve HTTP.

## Hands-On Lab

### Lab 3.1: Build a Simple FastAPI App Image

#### Step 1: Create Project Structure

```bash
mkdir -p myapp
cd myapp
```

#### Step 2: Create requirements.txt

```bash
cat > requirements.txt << 'EOF'
fastapi==0.104.1
uvicorn==0.24.0
EOF
```

#### Step 3: Create app.py

```bash
cat > app.py << 'EOF'
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from Docker"}

@app.get("/health")
async def health():
    return {"status": "ok"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
EOF
```

#### Step 4: Create Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

RUN useradd -m appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser app.py .

ENV PYTHONUNBUFFERED=1

EXPOSE 8000

USER appuser

HEALTHCHECK --interval=10s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

#### Step 5: Create .dockerignore

```bash
cat > .dockerignore << 'EOF'
__pycache__
*.pyc
.pytest_cache
.env
venv/
EOF
```

#### Step 6: Build the Image

```bash
docker build -t myapp:1.0.0 .
```

You should see output like:

```
[1/7] FROM python:3.12-slim
[2/7] WORKDIR /app
[3/7] RUN useradd -m appuser
[4/7] COPY requirements.txt .
[5/7] RUN pip install --no-cache-dir -r requirements.txt
[6/7] COPY --chown=appuser:appuser app.py .
[7/7] CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
=> exporting to image
=> => naming to docker.io/library/myapp:1.0.0
```

#### Step 7: Run the Container

```bash
docker run -d -p 8000:8000 --name myapp_run myapp:1.0.0
```

#### Step 8: Test the API

```bash
# Give it a second to start
sleep 2

# Test the root endpoint
curl http://localhost:8000/

# Test the health endpoint
curl http://localhost:8000/health
```

You should see:
```json
{"message":"Hello from Docker"}
{"status":"ok"}
```

#### Step 9: View Logs

```bash
docker logs -f myapp_run
```

You should see:
```
INFO:     Uvicorn running on http://0.0.0.0:8000
```

#### Step 10: Check Health

```bash
docker ps | grep myapp_run
```

The STATUS should include "healthy" after a few seconds.

#### Step 11: Stop and Clean Up

```bash
docker stop myapp_run
docker rm myapp_run
docker rmi myapp:1.0.0
```

### Lab 3.2: Rebuild with Changed Code

Modify `app.py`:

```python
@app.get("/")
async def root():
    return {"message": "Updated message"}
```

Rebuild:

```bash
docker build -t myapp:1.0.1 .
```

Notice: Layers 1-5 (FROM, WORKDIR, pip install) are cached and instant. Only layers 6-7 (COPY, CMD) rebuild.

### Lab 3.3: Multi-Stage Build Savings

Change your Dockerfile to include build tools:

```dockerfile
# Stage 1: Builder
FROM python:3.12 AS builder  # Full image, not slim!
WORKDIR /build
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app.py .
USER appuser
CMD ["python", "app.py"]
```

Compare image sizes:

```bash
docker build -t myapp:multi-stage .
docker images | grep myapp
```

The multi-stage image should be noticeably smaller.

## Cheat Sheet: Dockerfile Best Practices Checklist

Before shipping an image, verify:

- [ ] **Start with `-slim` base image** (not full OS)
- [ ] **Use multi-stage builds** to remove build tools from final image
- [ ] **Chain RUN commands** with `&&` to minimize layers
- [ ] **Put changeable things at the bottom** (COPY, CMD before COPY requirements)
- [ ] **Use .dockerignore** to exclude large files
- [ ] **Create a non-root user** (useradd, USER appuser)
- [ ] **Set PYTHONUNBUFFERED=1** for Python apps
- [ ] **Copy files with --chown** to set ownership
- [ ] **Use --no-cache-dir** with pip install to save space
- [ ] **Always EXPOSE your port** (documentation)
- [ ] **Include a HEALTHCHECK** for production apps
- [ ] **Use specific dependency versions** (not ==latest)
- [ ] **Don't include secrets** in the image (use env vars)
- [ ] **Set WORKDIR explicitly** (don't rely on /)

## Key Takeaways

- **FROM** starts with a base, **WORKDIR** sets the working directory
- **COPY** files from host, **RUN** executes commands
- **ENV** sets variables, **EXPOSE** documents ports
- **CMD** is the default command, **ENTRYPOINT** is fixed
- **Multi-stage** removes build bloat
- **Layer caching** makes rebuilds fast when you order instructions well
- **.dockerignore** speeds up builds by excluding large files

Now that you can build images, Module 4 teaches you to coordinate multiple containers with Docker Compose.
