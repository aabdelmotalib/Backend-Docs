# Module 1: What is Docker?

## The Analogy: Shipping Containers

Imagine you're shipping goods internationally. In the old days, customs agents had to open your cargo, inspect it, repackage it, and hope nothing broke during unloading. The same goods would be handled differently in different ports, causing chaos.

Then someone invented **shipping containers**—standardized, lockable boxes. Put your goods in a container in New York, and it arrives unchanged in Shanghai. The handling equipment is always the same. No repacking, no surprises.

**Docker containers work the same way for software.**

Before Docker, deploying an app meant:
- Installing Python 3.10 (or 3.11? 3.12?)
- Installing 47 dependencies with `pip`
- Configuring PostgreSQL
- Adding environment variables
- Praying it works on the production server

This is **fragile and manual**. Docker packages your entire application into a container—exact OS, exact Python version, exact dependencies, exact configuration—so it runs identically anywhere.

## The Problem Docker Solves

### "It Works on My Machine" Syndrome

You write a Python FastAPI app on your MacBook. It runs perfectly. You deploy it to Ubuntu on AWS, and it crashes. Why?

- Your MacBook has Python 3.12, the server has 3.10
- You installed a system dependency (`libpq-dev`) that's missing on the server
- Your `.env` file wasn't uploaded
- Permissions are different

**Docker solution**: Package your app with an exact Ubuntu base image, exact Python 3.12, exact dependencies. The container inherits all of this. It runs the same everywhere.

### Microservices Chaos

A real SaaS backend needs:
- API service (FastAPI)
- Worker service (Celery)
- PostgreSQL database
- Redis cache
- MinIO file storage

Without Docker, you'd need 5 different servers, or fight port conflicts and dependency hell on one server. With Docker, each service runs in its own isolated container, seeing its own filesystem but easily communicating with neighbors over a network.

## The Docker Architecture

Docker has three main components:

### 1. Docker Daemon (Server)

The **Docker daemon** is a background process that manages containers. It:
- Creates and runs containers
- Manages images
- Manages networks and volumes
- Enforces resource limits

It runs with root privileges because containers need to be isolated at the kernel level.

You interact with the daemon through the **Docker CLI**.

### 2. Docker CLI (Client)

When you type `docker run hello-world`, you're using the Docker CLI, which sends commands to the daemon.

```bash
docker run hello-world
    ↓ (REST API call)
Docker daemon: "Create and run a container from the hello-world image"
```

Think of it like `kubectl` for Kubernetes—the CLI is just the interface.

### 3. Docker Hub & Registries

Docker Hub is the app store for container images. It hosts millions of pre-built images:
- `ubuntu:24.04` — base Linux
- `python:3.12-slim` — Python runtime
- `postgres:15` — database
- `redis:7` — cache

When you run `docker pull postgres:15`, it downloads the image from Docker Hub to your machine.

## Real-World Example: Your SaaS Backend

In production, your PDF processing platform might look like this:

```
┌─────────────────────────────────────────────┐
│         Docker Host (AWS EC2)               │
├─────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐         │
│  │   API        │  │   Worker     │         │
│  │  Container   │  │  Container   │         │
│  │  Port 8000   │  │  (no port)   │         │
│  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐         │
│  │  PostgreSQL  │  │    Redis     │         │
│  │  Container   │  │  Container   │         │
│  │  Port 5432   │  │  Port 6379   │         │
│  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐         │
│  │    MinIO     │  │    Nginx     │         │
│  │  Container   │  │  Container   │         │
│  │  Port 9000   │  │  Port 80/443 │         │
│  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────┘
```

Each rectangle is an isolated container. They share the host's kernel but have their own:
- Filesystem
- Process space
- Network interfaces
- Resource limits

The **API container** talks to **PostgreSQL** and **Redis** containers using network names: `postgres:5432` and `redis:6379`. It doesn't matter what IP address they have—Docker's DNS resolves the name.

## Containers vs Virtual Machines

You might ask: "Isn't this just virtualization?"

**No.** Here's the critical difference:

### Virtual Machine Architecture

```
┌─────────────────────────────────┐
│         Your PC                 │
├─────────────────────────────────┤
│  Host OS: macOS                 │
│  Hypervisor (VirtualBox)        │
│  ├── Guest OS: Linux            │
│  │   ├── App 1                  │
│  │   └── App 2                  │
│  └── Guest OS: Linux            │
│      ├── App 3                  │
│      └── App 4                  │
└─────────────────────────────────┘
```

Each VM runs a **complete operating system**. VM 1 might be 2GB just for the OS.

### Container Architecture

```
┌─────────────────────────────────┐
│         Your PC                 │
├─────────────────────────────────┤
│  Host OS: Linux                 │
│  Docker daemon                  │
│  ├── Container 1 (30MB)         │
│  ├── Container 2 (30MB)         │
│  ├── Container 3 (30MB)         │
│  └── Container 4 (30MB)         │
│                                 │
│  All sharing the host kernel    │
└─────────────────────────────────┘
```

Containers **share the host's kernel**. They only package:
- Runtime (Python, Node.js, etc.)
- Dependencies
- Your code
- Configuration

A Python container might be 50MB. A PostgreSQL container is 200MB. That's it.

| Aspect | VM | Container |
|--------|----|-----------| 
| OS Overhead | 1-2 GB per VM | Shared kernel |
| Startup Time | 30-60 seconds | 100-500 ms |
| Isolation | Strong | Process-level |
| Density | 5-10 VMs on a server | 100+ containers |

**Why Docker works on Mac/Windows**: They run a lightweight Linux VM, then Docker containers run inside it. You don't notice the VM—Docker handles it transparently.

## How Docker Daemon Works (Simplified)

When you run a command:

```bash
docker run -d -p 8000:8000 --name api myapp:latest
```

Here's what happens:

1. **CLI parses the command** → "Run myapp:latest image, name it 'api', expose port 8000"
2. **CLI connects to daemon** → Sends REST API request
3. **Daemon checks if image exists** → If not, pulls from Docker Hub
4. **Daemon creates a container** → Allocates filesystem, network, PID space
5. **Daemon starts the process** → Runs the command specified in the image
6. **Daemon returns container ID** → `abc123def456...`

The `-d` flag means "detach"—run in the background. Without it, you'd see the container's output in your terminal.

!!! note
    Docker daemon runs as root, but individual containers can run as non-root users for security. This is a critical production best practice.

## Key Docker Concepts You'll Master

As you work through the next 6 modules, these concepts will become second nature:

- **Image** — A read-only blueprint for a container (like a recipe)
- **Container** — A running instance of an image (like a baked cake)
- **Layer** — Immutable parts of an image that get stacked (like cake layers)
- **Volume** — Persistent storage outside the container
- **Network** — How containers communicate with each other and the outside world
- **Registry** — A repository of images (Docker Hub, GitHub Container Registry)

## Hands-On Lab

### Lab 1.1: Install Docker and Run hello-world

**Goal**: Install Docker and understand basic container execution.

#### Step 1: Install Docker on Ubuntu

```bash
# Update package manager
sudo apt update

# Install Docker
sudo apt install -y docker.io

# Add your user to the docker group (so you don't need sudo every time)
sudo usermod -aG docker $USER

# Start the Docker daemon
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
docker --version
```

#### Step 2: Run hello-world

```bash
docker run hello-world
```

You'll see output like:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:abc123...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

#### Step 3: Understand Every Line of Output

Let's break it down:

| Output | Meaning |
|--------|---------|
| `Unable to find image 'hello-world:latest' locally` | You don't have it downloaded yet |
| `latest: Pulling from library/hello-world` | Downloading the latest version from Docker Hub |
| `2db29710123e: Pull complete` | One layer downloaded (this image is tiny—just 1 layer) |
| `Digest: sha256:abc123...` | Cryptographic hash of the image (ensures integrity) |
| `Status: Downloaded newer image...` | Download successful |
| `Hello from Docker!...` | Container started and printed this message |

#### Step 4: Verify the Image Was Downloaded

```bash
docker images
```

Output:
```
REPOSITORY    TAG       IMAGE ID      CREATED       SIZE
hello-world   latest    d2c94e258dcb  2 months ago  13.3kB
```

You now have a 13.3KB image on your system. That's the entire `hello-world` container!

#### Step 5: Run It Again (Much Faster)

```bash
docker run hello-world
```

This time it's instant—no download needed. The image is cached.

!!! tip
    The first run downloads the image. Subsequent runs use the cached copy. This is why your second `docker run` is 50x faster than the first.

### Lab 1.2: Run Ubuntu and Explore

```bash
# Run Ubuntu, dropping you into an interactive shell
docker run -it ubuntu:24.04 bash
```

Now you're inside a container! Try:

```bash
# See Linux version
cat /etc/os-release

# See what's installed
which python
which python3

# Exit the container
exit
```

**Key insight**: This is a minimal Ubuntu—no Python, no tools, just the OS basics. That's why containers are so lightweight. You only add what your app needs.

### Lab 1.3: Keep a Container Running

```bash
# Start a container that sleeps for 1 hour
docker run -d --name sleeper ubuntu:24.04 sleep 3600
docker ps  # Should show the sleeper container running
docker stop sleeper  # Stop it gracefully
docker rm sleeper  # Delete it
```

**Key insight**: `docker ps` shows running containers. `docker ps -a` shows all containers (running + stopped).

## Cheat Sheet: Top 10 Docker Commands

| Command | What it Does |
|---------|-------------|
| `docker pull IMAGE` | Download an image from Docker Hub |
| `docker images` | List all images on your system |
| `docker run IMAGE` | Create and start a container from an image |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (running + stopped) |
| `docker stop CONTAINER` | Stop a running container gracefully |
| `docker kill CONTAINER` | Force-stop a container immediately |
| `docker rm CONTAINER` | Delete a stopped container |
| `docker logs CONTAINER` | View container output/logs |
| `docker exec CONTAINER COMMAND` | Run a command inside a running container |

**Remember**: The difference between `stop` (graceful, SIGTERM) and `kill` (forceful, SIGKILL) matters in production.

Now let's move to Module 2 and understand what images actually are.
