# Module 7: PostgreSQL in Production

## The Analogy: Moving from Home to Office

Home (development):
- 1 laptop, 1 database
- If hard drive fails, you lose everything
- No backup power
- You manage everything

Office (production):
- Multiple servers, replicas, failover
- Nightly backups stored off-site
- UPS (power backup)
- IT team monitors 24/7

## Connection Pooling

### The Problem

Every request creates a connection:

```
FastAPI Worker 1 → PostgreSQL
FastAPI Worker 2 → PostgreSQL
FastAPI Worker 3 → PostgreSQL
...
FastAPI Worker 100 → PostgreSQL

100 connections. Database is overwhelmed.
```

### The Solution: Connection Pool

```
FastAPI Workers → Connection Pool → PostgreSQL (10 connections)
```

A pool reuses connections:

1. Request 1 gets connection from pool
2. Request 1 finishes, returns connection to pool
3. Request 2 reuses the same connection
4. Max 10 connections total, can handle 1000 requests

### PgBouncer (External Pool)

```
FastAPI (connections) → PgBouncer (pool) → PostgreSQL
```

PgBouncer is a standalone connection pooler:

```ini
# pgbouncer.ini
[databases]
myapp = host=postgres port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

FastAPI connects to PgBouncer:

```python
DATABASE_URL = "postgresql+asyncpg://user:pass@pgbouncer:6432/myapp"
```

### SQLAlchemy Pool (In-App)

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://...",
    pool_size=20,           # Keep 20 connections open
    max_overflow=10,        # Allow 10 extra if needed
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True      # Test connection before using
)
```

Settings:
- `pool_size` — connections to keep open
- `max_overflow` — extra connections if pool exhausted
- `pool_recycle` — close/reopen connections periodically (prevent timeout)
- `pool_pre_ping` — test connection before using (prevents "connection closed" errors)

## Backup Strategy

### Full Backup

```bash
# Daily backup
docker exec postgres pg_dump -U postgres mydb | gzip > backup_$(date +%Y%m%d).sql.gz

# Restore from backup
gunzip -c backup_20240115.sql.gz | docker exec -i postgres psql -U postgres mydb
```

### Point-in-Time Recovery (PITR)

PostgreSQL writes transaction logs (WAL) to disk. Allows recovery to any moment in time.

Enable in PostgreSQL:

```sql
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 3;
```

Backup both database and WAL:

```bash
# Full backup + WAL
pg_basebackup -D /backup/data -Ft -z -P

# Later: restore to specific time
pg_restore -d mydb /backup/data/base.tar.gz
# Then replay WAL up to timestamp
```

## Replication

### Master-Replica Setup

```
Master PostgreSQL (writes)
    ↓
Replica PostgreSQL (reads)
    ↓
Replica PostgreSQL (reads)
```

Benefits:
- **High availability** — if master fails, promote replica
- **Read scaling** — distribute SELECT queries to replicas
- **Backup source** — replicas are live backups

Configuration:

**Master** (`postgresql.conf`):

```ini
wal_level = replica
max_wal_senders = 10
```

**Replica**:

```bash
pg_basebackup -h master_ip -D /var/lib/postgresql/data -U replication
```

Then point your replicas to master:

```
FastAPI (reads) → Load Balancer → Replica 1 (read-only)
                                   Replica 2 (read-only)
FastAPI (writes) → Master (writable)
```

## Monitoring

### Essential Metrics

```sql
-- Active connections
SELECT count(*) FROM pg_stat_activity;

-- Cache hit ratio (target: >99%)
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;

-- Slow queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Index usage
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Unused
ORDER BY idx_scan;
```

### Monitoring Tools

- **pg_stat_statements** — track slow queries
- **pgAdmin** — web UI for management
- **Prometheus + Grafana** — dashboards and alerts
- **DataDog/New Relic** — full monitoring

### Health Check Endpoint (FastAPI)

```python
@app.get("/health/db")
async def db_health(session: AsyncSession = Depends(get_session)):
    """Health check: can we connect to database?"""
    try:
        await session.execute(text("SELECT 1"))
        return {"database": "healthy"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))
```

## Docker Compose for Production

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - app_network

  pgbouncer:
    image: pgbouncer:latest
    environment:
      DATABASES_HOST: postgres
      DATABASES_PORT: 5432
      DATABASES_USER: ${POSTGRES_USER}
      DATABASES_PASSWORD: ${POSTGRES_PASSWORD}
      DATABASES_DBNAME: ${POSTGRES_DB}
    ports:
      - "6432:6432"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app_network

  api:
    build: .
    environment:
      DATABASE_URL: postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@pgbouncer:6432/${POSTGRES_DB}
    depends_on:
      - pgbouncer
    networks:
      - app_network

volumes:
  postgres_data:

networks:
  app_network:
```

## Deployment Checklist

Before go-live:

- [ ] Connection pooling configured
- [ ] Backup automation set up (daily + off-site)
- [ ] Replication tested (master-replica)
- [ ] Indexes on hot queries (use EXPLAIN ANALYZE)
- [ ] Slow query log enabled
- [ ] Monitoring alerts configured
- [ ] Disaster recovery plan documented
- [ ] Database password rotated monthly
- [ ] Point-in-time recovery tested
- [ ] Capacity planning done (can scale to 10x users?)

## Real PDF SaaS Production Config

```python
# config.py
from pydantic import BaseSettings
import os
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.asyncio import AsyncSession

class Settings(BaseSettings):
    # Database
    database_url: str = os.getenv("DATABASE_URL")
    database_pool_size: int = 20
    database_max_overflow: int = 10
    database_pool_recycle: int = 3600
    
    class Config:
        env_file = ".env"

settings = Settings()

# Create engine with pooling
engine = create_async_engine(
    settings.database_url,
    pool_size=settings.database_pool_size,
    max_overflow=settings.database_max_overflow,
    pool_recycle=settings.database_pool_recycle,
    pool_pre_ping=True,
    echo=False
)

async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_session():
    async with async_session() as session:
        yield session
```

## Hands-On Lab

### Lab 7.1: Create Backup Script

Create `backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/backups"
POSTGRES_CONTAINER="postgres"
POSTGRES_USER="postgres"
POSTGRES_DB="mydb"

# Create backup
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.sql.gz"

docker exec $POSTGRES_CONTAINER pg_dump -U $POSTGRES_USER $POSTGRES_DB | gzip > $BACKUP_FILE

echo "Backup created: $BACKUP_FILE"

# Keep only last 7 days
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete
```

Run:

```bash
chmod +x backup.sh
./backup.sh
```

### Lab 7.2: Test Restore

```bash
# Simulate data loss
docker exec postgres psql -U postgres -d mydb -c "DROP TABLE users;"

# Restore from backup
gunzip -c /backups/backup_20240115_120000.sql.gz | \
  docker exec -i postgres psql -U postgres -d mydb

# Verify
docker exec postgres psql -U postgres -d mydb -c "\dt"
```

### Lab 7.3: Monitor Queries

```sql
-- Create pg_stat_statements extension
CREATE EXTENSION pg_stat_statements;

-- Run some queries
SELECT * FROM users WHERE id = 1;
SELECT * FROM subscriptions WHERE active = TRUE;

-- See slowest
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 5;
```

## Cheat Sheet: Production PostgreSQL

### Connection Pooling

```python
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_recycle=3600,
    pool_pre_ping=True
)
```

### Backup

```bash
pg_dump -U postgres mydb | gzip > backup.sql.gz
gunzip -c backup.sql.gz | psql -U postgres mydb
```

### Monitor Connections

```sql
SELECT count(*) FROM pg_stat_activity;
```

### Monitor Cache Hit Ratio

```sql
SELECT 
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;
```

### Slow Queries

```sql
SELECT query, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;
```

## Key Takeaways

- **Connection pooling** prevents database overload
- **Backups are essential** — automate daily, store off-site
- **Replication provides high availability** — master-replica setup
- **Monitoring detects issues early** — track connections, cache hits, slow queries
- **PITR = disaster recovery** — restore to any moment in time
- **Health checks** — API knows if database is healthy
- **Capacity planning** — predict growth, scale before problems

You now have complete PostgreSQL mastery covering fundamentals through production deployment.

Next: Redis for caching and Celery task queue.
