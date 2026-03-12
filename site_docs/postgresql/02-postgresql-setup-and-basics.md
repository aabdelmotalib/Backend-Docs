# Module 2: PostgreSQL Setup and Basics

## The Analogy: Installing a New Workshop

Installing PostgreSQL is like setting up a new workshop:

1. **Acquire the tools** — Download PostgreSQL
2. **Set up the space** — Create data directory, initialize database
3. **Open for business** — Start the server
4. **Test everything** — Connect and make sure it works

## Installation Options

### Option 1: Docker (Recommended)

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15-alpine
```

This:
- Downloads PostgreSQL 15-alpine image
- Sets password to `yourpassword`
- Creates default database `myapp`
- Exposes on localhost:5432
- Stores data in Docker volume (persistent)

Check it's running:

```bash
docker ps | grep postgres
```

### Option 2: Homebrew (macOS)

```bash
brew install postgresql@15
brew services start postgresql@15
```

### Option 3: Native Installation (Linux)

```bash
sudo apt-get install postgresql postgresql-contrib
sudo systemctl start postgresql
```

## Connection Strings

A **connection string** tells a client how to connect to PostgreSQL.

Format: `postgresql://user:password@host:port/database`

### Examples

```
postgresql://postgres:password@localhost:5432/myapp
postgresql://user:pass@db.example.com:5432/production
postgresql+asyncpg://user:pass@localhost/myapp  (async in FastAPI)
```

Parts:
- `user` — login name (default: `postgres`)
- `password` — password (set during installation)
- `host` — server address (default: `localhost`)
- `port` — port number (default: `5432`)
- `database` — database name (default: `postgres`)

## psql: The PostgreSQL CLI

**psql** is the command-line interface for PostgreSQL. Using it, you:
- Connect to a database
- Run SQL queries
- List tables, columns, constraints
- Backup and restore data

### Connect Locally (Docker)

```bash
# Connect as postgres user to myapp database
docker exec -it postgres psql -U postgres -d myapp

# Interactive prompt appears
myapp=#
```

### Common psql Commands

While in psql prompt:

```
\l          List all databases
\c dbname   Connect to database
\dt         List tables
\d tablename   Describe table
\du         List users
\h COMMAND  Help on SQL command
\q          Quit psql
\i file.sql Execute SQL file
\x          Toggle expanded display (better for wide rows)
```

### Run SQL from Command Line

```bash
# Single query
docker exec postgres psql -U postgres -d myapp -c "SELECT * FROM users;"

# From file
docker exec postgres psql -U postgres -d myapp -f schema.sql

# From stdin
echo "SELECT version();" | docker exec -i postgres psql -U postgres -d myapp
```

## Creating the PDF SaaS Database

### Step 1: Connect

```bash
docker exec -it postgres psql -U postgres
```

### Step 2: Create Database

```sql
CREATE DATABASE pdf_saaS;

\l  -- List databases
```

### Step 3: Connect to Database

```sql
\c pdf_saaS
```

### Step 4: Create Schema

Create the core tables:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan VARCHAR(50) NOT NULL,
    pages_per_month INTEGER NOT NULL,
    price_cents INTEGER NOT NULL,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);

CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    subscription_id INTEGER NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    status VARCHAR(50) DEFAULT 'queued',
    input_url TEXT NOT NULL,
    output_format VARCHAR(10),
    page_range VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    error_message TEXT
);

CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    subscription_id INTEGER NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    amount_cents INTEGER NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (amount_cents > 0)
);

CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Step 5: Verify

```sql
\dt                          -- List tables
\d users                     -- Describe users table
SELECT * FROM users;         -- Empty table
```

## Backing Up and Restoring

### Backup (pg_dump)

Backup entire database:

```bash
docker exec postgres pg_dump -U postgres pdf_saaS > backup.sql
```

Backup only schema (no data):

```bash
docker exec postgres pg_dump -U postgres --schema-only pdf_saaS > schema.sql
```

Backup as binary format (faster for large databases):

```bash
docker exec postgres pg_dump -U postgres --format=custom pdf_saaS > backup.dump
```

### Restore

From SQL file:

```bash
docker exec -i postgres psql -U postgres pdf_saaS < backup.sql
```

From binary dump:

```bash
docker exec postgres pg_restore -U postgres --dbname=pdf_saaS backup.dump
```

## Connection from Python

### Using asyncpg (async driver for FastAPI)

```python
import asyncpg

# Connect
conn = await asyncpg.connect(
    user='postgres',
    password='password',
    database='pdf_saaS',
    host='localhost'
)

# Query
rows = await conn.fetch('SELECT * FROM users')

# Insert
await conn.execute(
    'INSERT INTO users (email, password_hash) VALUES ($1, $2)',
    'alice@example.com', 'hashed_password'
)

# Close
await conn.close()
```

### Using SQLAlchemy (ORM, covered in Module 3)

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("postgresql+asyncpg://postgres:password@localhost/pdf_saaS")

async with engine.begin() as conn:
    result = await conn.execute(text("SELECT * FROM users"))
    rows = result.fetchall()
```

## Docker Compose Setup

For the full SaaS stack:

**docker-compose.yml**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: pdf_saaS
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./schema.sql:/docker-entrypoint-initdb.d/schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:dev_password@postgres:5432/pdf_saaS
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

Run:

```bash
docker-compose up -d
```

Access database from API:

```bash
docker exec -i postgres psql -U postgres -d pdf_saaS
```

## Hands-On Lab

### Lab 2.1: Set Up PostgreSQL

```bash
# Start PostgreSQL
docker run -d \
  --name postgres_lab \
  -e POSTGRES_PASSWORD=labpass \
  -e POSTGRES_DB=test_db \
  -p 5432:5432 \
  postgres:15-alpine

# Wait for it to start
sleep 5

# Connect
docker exec -it postgres_lab psql -U postgres -d test_db
```

### Lab 2.2: Create and Populate Tables

In psql:

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO posts (title, content) VALUES
  ('First Post', 'Hello World'),
  ('Second Post', 'More content'),
  ('Third Post', 'Even more');

SELECT * FROM posts;
```

### Lab 2.3: Backup and Restore

From terminal:

```bash
# Backup
docker exec postgres_lab pg_dump -U postgres test_db > test_db.sql

# View backup
head -20 test_db.sql

# Drop table
docker exec -it postgres_lab psql -U postgres -d test_db -c "DROP TABLE posts;"

# Restore
docker exec -i postgres_lab psql -U postgres -d test_db < test_db.sql

# Verify
docker exec postgres_lab psql -U postgres -d test_db -c "SELECT * FROM posts;"
```

## Cheat Sheet: PostgreSQL Setup

### Docker Startup

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15-alpine
```

### Connect via psql

```bash
docker exec -it postgres psql -U postgres -d mydb
```

### Connection String

```
postgresql+asyncpg://postgres:password@localhost:5432/mydb
```

### Common psql Commands

```
\l           List databases
\c dbname    Connect to database
\dt          List tables
\d table     Describe table
\du          List users
\q           Quit
```

### Backup/Restore

```bash
docker exec postgres pg_dump -U postgres mydb > backup.sql
docker exec -i postgres psql -U postgres mydb < backup.sql
```

## Key Takeaways

- **Docker is the easiest way to run PostgreSQL** — no OS installation
- **psql is the command-line tool** — learn its commands
- **Connection string = complete address** — user:pass@host:port/db
- **Create database first, then connect** — \c dbname
- **Backup with pg_dump** — never lose data
- **Restore with psql** — recover from mistakes
- **asyncpg for async code** — use with FastAPI

Module 3 teaches SQLAlchemy ORM — writing Python instead of SQL.
