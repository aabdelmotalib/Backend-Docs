# Module 8: What Breaks and How to Fix It

## The Top 15 Production Problems

You'll encounter these. Here's exactly how to diagnose and fix.

---

## 1. Container is Unhealthy or Keeps Restarting

### Symptoms
- `docker ps` shows STATUS: "Restarting (1)" or "Unhealthy"
- Service goes down, comes back, goes down again (5-second cycle)
- Users see 502 Bad Gateway

### Diagnosis

```bash
# Check container status
docker ps | grep api

# View logs (last 100 lines)
docker logs api --tail 100

# Check why it exited
docker logs api | grep -i error | tail -20

# Specific errors to look for:
#   "Connection refused" → database not running
#   "ModuleNotFoundError" → missing dependency
#   "MemoryError" → OOM (out of memory)
#   "Permission denied" → cannot write to volume
#   "Address already in use" → port conflict
```

### Fix

**If database connection error**:
```bash
# Check PostgreSQL
docker ps | grep postgres

# If not running
docker start postgresql

# Verify connectivity from api container
docker exec api psql -h postgresql -U pdf_user -d pdf_db -c "SELECT 1"
```

**If out of memory**:
```bash
# Check RAM usage
docker stats api

# Increase container memory in docker-compose.yml
# memory: 512M → memory: 1024M

# Restart
docker compose down && docker compose up -d
```

**If missing dependencies**:
```bash
# Rebuild image
docker build -t pdf-api:latest .

# Restart
docker compose up -d api
```

---

## 2. API Returns 502 Bad Gateway

### Symptoms
- Browser shows "502 Bad Gateway" or "Temporary Error"
- `curl https://pdf-platform.example.com/upload` returns 502
- Nginx shows "upstream prematurely closed connection"

### Diagnosis

```bash
# Check if FastAPI is running
curl http://localhost:8000/health
# If connection refused → FastAPI is down

# Check FastAPI logs
docker logs api --tail 50

# Check Nginx logs
docker logs nginx | grep 502 | tail -20

# Check if port 8000 is listening
docker exec nginx netstat -tuln | grep 8000

# Check Docker network
docker network inspect pdf-platform_default
# Verify 'api' container is connected
```

### Fix

**If API container crashed**:
```bash
# Restart
docker restart api

# Wait for it to be healthy
sleep 10
curl http://localhost:8000/health
```

**If FastAPI hanging**:
```bash
# Check for hung processes
docker exec api ps aux

# Kill hung process
docker exec api pkill -9 python

# Or restart container
docker restart api
```

**If Nginx can't reach API**:
```bash
# Verify container name in nginx.conf
cat /etc/nginx/conf.d/default.conf | grep upstream

# Should show: proxy_pass http://api:8000;
# If shows 'localhost' or '127.0.0.1', it won't work!

# Fix: Use Docker network name
docker exec nginx cat /etc/nginx/conf.d/default.conf
# Look for: upstream fastapi { server api:8000; }
```

---

## 3. Database Connection Pool Exhausted

### Symptoms
- Errors: `too many connections` or `pool timeout`
- 503 Service Unavailable
- Works with few users, breaks under load

### Diagnosis

```bash
# Check current connections in PostgreSQL
docker exec postgresql psql -U pdf_user -d pdf_db -c "SELECT count(*) FROM pg_stat_activity;"

# Check max allowed
docker exec postgresql psql -U pdf_user -d pdf_db -c "SHOW max_connections;"
# Might show: 100

# Check to see what's connected
docker exec postgresql psql -U pdf_user -d pdf_db -c "SELECT * FROM pg_stat_activity WHERE state!='idle';"
```

### Fix

**Increase pool size**:
```bash
# Edit .env
nano .env
# Change: DATABASE_POOL_SIZE=10 → DATABASE_POOL_SIZE=30

# Restart api
docker restart api
```

**Increase PostgreSQL max connections**:
```bash
# Edit postgresql.conf
docker exec postgresql psql -U pdf_user -d pdf_db -c "ALTER SYSTEM SET max_connections = 200;"

# Restart PostgreSQL
docker restart postgresql
```

**Find connection leaks**:
```bash
# Connections that are idle for > 1 hour
docker exec postgresql psql -U pdf_user -d pdf_db -c "
SELECT now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle'
AND now() - query_start > interval '1 hour';
"

# Kill idle connections
docker exec postgresql psql -U pdf_user -d pdf_db -c "
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
AND now() - query_start > interval '1 hour';
"
```

---

## 4. Celery Worker Not Picking up Tasks (Queue Growing)

### Symptoms
- Users upload files, but conversion never starts
- Redis queue growing: `redis-cli LLEN celery` shows 100+ messages
- Jobs stuck at status="queued" for hours

### Diagnosis

```bash
# Is worker running?
docker ps | grep worker

# If not running
docker start worker

# If running, check its logs
docker logs worker --tail 50

# Check Redis queue
docker exec redis redis-cli LLEN celery

# Check if worker is consuming
docker logs worker | grep "Received task\|Executing"
# No output? Worker not consuming
```

### Fix

**Restart worker**:
```bash
docker restart worker
```

**Flush problematic queue and restart**:
```bash
# Flush queue
docker exec redis redis-cli FLUSHDB

# Restart worker
docker restart worker

# Jobs are lost, but at least worker is responsive
# Users should re-upload
```

**Check for worker crashes**:
```bash
# Look for errors in logs
docker logs worker | grep -i "error\|exception" | tail -20

# If memoryError or timeout
# Increase worker RAM in docker-compose.yml

# If specific task function missing
# Check tasks.py is up-to-date
```

---

## 5. LibreOffice Conversion Failing Silently

### Symptoms
- Job status never changes from "converting"
- Worker logs show LibreOffice started but no error
- Stalled for 30+ minutes

### Diagnosis

```bash
# Check LibreOffice process
docker exec worker ps aux | grep soffice
# If no output: didn't start

# Check LibreOffice logs (if available)
docker logs worker | grep -i "libreoffice\|soffice"

# Check if LibreOffice container running (if separate)
docker ps | grep libreoffice

# Check disk space (LibreOffice needs temp space)
docker exec worker df -h
# If / is 99% full: out of space

# Check PDF is actually valid
file /tmp/test.pdf
# Should show "PDF Document" not "empty"
```

### Fix

**Restart LibreOffice**:
```bash
# If separate container
docker restart libreoffice

# Or restart worker (will respawn)
docker restart worker
```

**Increase timeout**:
```bash
# In tasks.py, increase timeout
@celery_app.task(bind=True, time_limit=120)  # 2 minutes
def convert_pdf_task(self, job_id, ...):
    # subprocess.run(..., timeout=60)
```

**Free up disk space**:
```bash
# Find large files
du -sh /var/lib/docker/volumes/*/

# Delete temp conversions
docker exec worker rm -rf /tmp/*.pdf /tmp/*.png

# Or rebuild containers (more drastic)
docker system prune -a
```

---

## 6. MinIO Unreachable

### Symbols
- Errors: `Connection refused` or `Cannot reach MinIO`
- File upload fails, but database insert succeeds
- 500 error on `/upload` endpoint

### Diagnosis

```bash
# Is MinIO running?
docker ps | grep minio

# Is it listening on port 9000?
docker exec minio netstat -tuln | grep 9000

# Can API reach it?
docker exec api curl -I http://minio:9000/minio/health/live

# Check MinIO health
docker logs minio | tail -50
```

### Fix

**Restart MinIO**:
```bash
docker restart minio
```

**Check disk space**:
```bash
# MinIO's data volume
du -sh /var/lib/docker/volumes/pdf_minio_data

# If near limit (e.g., VPS is 160GB)
# Delete old files or expand volume
```

**Check credentials**:
```bash
# In .env, verify MinIO credentials match
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123

docker compose down && docker compose up -d minio
```

---

## 7. ClamAV Not Responding

### Symptoms
- Upload hangs for 60+ seconds
- Error: `Connection timed out` when calling ClamAV
- Files not being scanned

### Diagnosis

```bash
# Is ClamAV running?
docker ps | grep clamav

# Check ClamAV socket
docker exec clamav netstat -tuln | grep 3310

# Try to connect
docker exec api clamdscan --version
# If fails: socket not responding

# Check logs
docker logs clamav | tail -30
```

### Fix

**Restart ClamAV**:
```bash
docker restart clamav

# Wait for it to load signature database (1-2 min)
sleep 120
```

**Update signatures manually**:
```bash
# ClamAV updates virus definitions periodically
docker exec clamav freshclam

# Or let it run automatically (usually enabled)
```

**Skip ClamAV if non-critical**:
```python
# In tasks.py, wrap in try-except
try:
    is_infected = await clam.scan_stream(file.file)
except timeout:
    logger.warning("ClamAV timeout, skipping scan")
    is_infected = False  # Safe default
```

---

## 8. Redis Out of Memory

### Symptoms
- Errors: `OOM command not allowed when used memory > maxmemory`
- Rate limiting stops working
- Session timers broken

### Diagnosis

```bash
# Check Redis memory usage
docker exec redis redis-cli INFO memory | grep -E "used_memory|maxmemory"

# Output:
# used_memory_human:256M
# maxmemory:512M

# If used > 95% of max: OOM imminent

# Check what's taking space
docker exec redis redis-cli --bigkeys
# Shows large keys
```

### Fix

**Increase maxmemory**:
```bash
# In docker-compose.yml
command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru

# Restart
docker restart redis
```

**Enable eviction**:
```bash
# Automatically delete least-used keys when full
command: redis-server --maxmemory-policy allkeys-lru

# LRU = Least Recently Used
# When memory full, oldest unused keys deleted
```

**Identify memory hogs**:
```bash
# Find users with most sessions
docker exec redis redis-cli KEYS "session:*:start" | wc -l
# Shows count of active sessions

# Find stuck keys (not expiring)
docker exec redis redis-cli KEYS "*" | head -100
docker exec redis redis-cli TTL keyname
# If -1, no expiry (memory leak!)
```

---

## 9. Paymob Webhook Not Arriving

### Symptoms
- User pays on Paymob, payment succeeds
- But subscription not created (frontend still shows "Buy Plan")
- Webhook never logged in FastAPI

### Diagnosis

```bash
# Check if webhook was received
docker logs api | grep "webhook" | tail -20

# Check if FastAPI is handling it
docker logs api | grep "payments" | tail -20

# Verify webhook URL in code
grep -r "payments/webhook" .
# Should match Paymob dashboard settings

# Check if signature validates
docker logs api | grep "HMAC\|signature" | grep -i fail
```

### Fix

**Verify webhook URL in Paymob**:
1. Paymob Dashboard → Settings → Developer Settings
2. Webhook URL should be: `https://pdf-platform.example.com/payments/webhook`
3. Not: `http://` (must be HTTPS)
4. Not: `localhost` or IP address

**Test webhook manually**:
```bash
# Send test webhook from Paymob dashboard
# Settings → Webhooks → Test

# Check if FastAPI received it
docker logs api | grep -i "payment\|webhook"
```

**Check signature secret**:
```bash
# In .env, verify API key matches
PAYMOB_API_KEY=$(cat .env | grep PAYMOB_API_KEY)

# Signature calculated as:
# HMAC-SHA512(webhook_body, PAYMOB_API_KEY)

# If API key wrong, signature validation fails
```

**Manual payment sync** (workaround):
```python
# Run this to find unpaid orders and mark as paid
from models import Payment, Subscription

payments = db.query(Payment).filter(Payment.status == "pending").all()

for p in payments:
    # Query Paymob API to check actual status
    resp = requests.get(f"https://accept.paymob.com/api/orders/{p.order_id}", ...)
    if resp['status'] == "paid":
        p.status = "paid"
        create_subscription_from_payment(p)

db.commit()
```

---

## 10. SSL Certificate Expired

### Symptoms
- Browser shows "Certificate Expired" warning
- Users can't access site (browsers refuse to connect)
- HTTPS redirects broken

### Diagnosis

```bash
# Check certificate expiry
certbot certificates

# Or check directly
openssl s_client -connect pdf-platform.example.com:443 2>/dev/null | grep "Not After"

# If shows date in the past: EXPIRED
```

### Fix

**Renew certificate immediately**:
```bash
certbot renew --force-renewal

# Restart Nginx
docker exec nginx nginx -s reload

# Verify
curl -I https://pdf-platform.example.com
# Should show 200, not SSL error
```

**Automate renewal**:
```bash
# Certbot already set up auto-renewal (usually)
# Check:
systemctl status certbot.timer

# If not running, enable:
systemctl enable certbot.timer
systemctl start certbot.timer

# This runs renewable checks every day
```

---

## 11. Disk Space Full (MinIO Data)

### Symptoms
- Upload fails with "no space left on device"
- Docker services crash due to no available space
- `df -h` shows 100% usage

### Diagnosis

```bash
# Check disk usage
df -h

# Find largest directories
du -sh /* | sort -hr | head -10

# MinIO data location
du -sh /var/lib/docker/volumes/pdf_minio_data/

# Check which PDFs are taking space (old uploads)
# MinIO UI at http://localhost:9000/
```

### Fix

**Delete old files**:
```bash
# Via MinIO console
# Go to http://localhost:9000
# Login: minioadmin / minioadmin
# Navigate to uploads bucket
# Delete old input files (pre-conversion)

# Or via CLI
docker exec minio mc rm minio/uploads/input-files --recursive --force

# Keep output files (user downloads)
```

**Expand volume**:
```bash
# If Hetzner, add another volume
# Click VPS → Storage → Add Volume
# Attach to VPS

# Mount on server
# On Hetzner:
hcloud volume attach <volume_id> <server_id>

# Then mount in filesystem
mkdir /mnt/data2
mkfs.ext4 /dev/<device>
mount /dev/<device> /mnt/data2

# Move MinIO data
docker stop minio
mv /var/lib/docker/volumes/pdf_minio_data /mnt/data2/
# Re-create symlink or update docker-compose paths
```

---

## 12. Alembic Migration Failed Mid-Way

### Symptoms
- Tried to run `alembic upgrade head`
- Migration on step 3-of-5 failed
- Database state is inconsistent (partially migrated)

### Diagnosis

```bash
# Check migration history
docker exec api alembic history

# See which migrations ran
docker exec api alembic current
# Shows: alembic_version = 2024_01_12_001

# Check failed migration
docker logs api | grep -i "alembic\|migration" | tail -30
```

### Fix

**Downgrade to last working version**:
```bash
# Downgrade one step
docker exec api alembic downgrade -1

# Or downgrade to specific revision
docker exec api alembic downgrade 2024_01_11_005
```

**Fix the migration script and retry**:
```bash
# Edit the failing migration
nano alembic/versions/2024_01_12_001_add_column.py

# Fix the SQL (maybe typo in column name, etc.)

# Retry
docker exec api alembic upgrade head
```

**If critical, backup and restore**:
```bash
# Backup current database
docker exec postgresql pg_dump -U pdf_user pdf_db | gzip > backup.sql.gz

# Restore to clean state
docker exec postgresql psql -U pdf_user -d pdf_db < clean_backup.sql
# (Requires clean backup from before the failed migration)

# Then retry migration
```

---

## 13. JWT Tokens Rejected (Clock Skew)

### Symptoms
- Login works, but then 401 errors: "Invalid token" or "Token expired"
- Other changes, same error
- Works fine locally, fails in production

### Diagnosis

```bash
# Check server time
docker exec api date

# Check FastAPI clock
docker exec api python3 -c "from datetime import datetime; print(datetime.now())"

# Check if off from actual time (NTP sync issue)
ntpdate -q time.nist.gov
```

### Fix

**Sync clock**:
```bash
# Enable NTP daemon
apt install ntpdate -y
ntpd -u ntp:ntp
systemctl restart ntpd

# Or manually sync
date -s "$(curl -s https://www.google.com | grep -oP 'Date: \K.*' | head -c 25)"
```

**Increase token TTL**:
```python
# In settings.py
ACCESS_TOKEN_EXPIRE_MINUTES = 30  # Was 10

# Gives more buffer for clock skew
```

**Invalidate all sessions (nuclear option)**:
```bash
# Clear JWT blacklist
docker exec redis redis-cli DEL jwt:blacklist:*

# Force users to re-login
# (old tokens treated as valid again)
```

---

## 14. Rate Limiter Blocking Legitimate Traffic

### Symptoms
- Users get 429 Too Many Requests
- It's not bot traffic (real users uploading)
- Happens at peak times

### Diagnosis

```bash
# Check rate limit config
grep -r "limit_req" /etc/nginx/

# Check Redis rate limit counters
docker exec redis redis-cli KEYS "ratelimit:*" | head -20

# See the counter for a specific IP
docker exec redis redis-cli GET "ratelimit:ip:203.0.113.45:count"
```

### Fix

**Increase rate limit**:
```bash
# In /etc/nginx/nginx.conf (or Dockerfile nginx config)
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

# Change to:
limit_req_zone $binary_remote_addr zone=api:10m rate=200r/s;

# Or exclude certain IPs:
# geo module to whitelist office IPs
```

**Implement per-user rate limit** (better):
```python
# Instead of IP-based, rate limit by user_id
@limiter.limit("100 per minute")  # Per user, not per IP
def upload():
    ...
```

**Burst capacity**:
```nginx
# Allow bursts (e.g., 10 requests over 1 second)
limit_req zone=api burst=10 nodelay;
```

---

## 15. Memory Leak in Worker Container (OOM Kill Loop)

### Symptoms
- Worker starts, processes tasks, then memory usage grows
- Eventually: "Killed" (OOM killer terminated the process)
- Docker restarts container, repeats

### Diagnosis

```bash
# Monitor memory over time
docker stats worker --no-stream
# Run multiple times, see if RSS grows

# Check container limits
docker inspect worker | grep -i memory

# Look for memory-related errors
docker logs worker | grep -i "memory\|oom"
```

### Fix

**Identify memory leak** (in tasks.py):
```python
# Common leak: session not closed
db = Session()
db.query(...)
# No db.close()!

# Fix:
def convert_pdf_task(...):
    db = SessionLocal()
    try:
        # ... conversion code
    finally:
        db.close()

# Or use context manager:
with Session() as db:
    # ... code
    # Auto-closes
```

**Increase container memory**:
```yaml
# docker-compose.yml
services:
  worker:
    memory: 512M  # Change to 1G
```

**Restart worker periodically**:
```bash
# Schedule restart every 8 hours
# In crontab:
0 */8 * * * docker restart worker

# Clears leaked memory
```

**Profile memory usage**:
```python
# Add memory profiler to tasks.py
from memory_profiler import profile

@profile
def convert_pdf_task(...):
    # Now see line-by-line memory usage
    ...
```

---

## Debugging Checklist

When something breaks in production:

1. **Check logs first**
   ```bash
   docker logs <container> --tail 100 | grep -i error
   ```

2. **Check service status**
   ```bash
   docker ps  # All containers running?
   docker compose ps
   ```

3. **Check network connectivity**
   ```bash
   docker exec api ping redis  # Can services reach each other?
   ```

4. **Check resources**
   ```bash
   docker stats  # CPU, RAM, disk?
   df -h        # Disk full?
   ```

5. **Check configurations**
   ```bash
   cat .env  # All secret keys present?
   grep -r "localhost" docker-compose.yml  # Network issues?
   ```

6. **Check databases**
   ```bash
   docker exec postgresql psql -U pdf_user -d pdf_db -c "SELECT COUNT(*) FROM users;"
   docker exec redis redis-cli PING
   ```

7. **Restart services systematically**
   ```bash
   docker restart <failing-service>
   # Wait 30 seconds
   # Retry operation
   # Still broken? Restart all
   docker compose restart
   ```

8. **Review recent changes**
   ```bash
   git log --oneline -10
   # What deployed recently? Did it cause this?
   ```

---

## Key Takeaway

Most production issues fall into these categories:

1. **Service crashed** → restart
2. **Resource exhausted** → increase limits
3. **Dependency unreachable** → restart dependency
4. **Configuration wrong** → check .env
5. **Code bug** → check logs, fix, redeploy
6. **Database issue** → check connections, migrations
7. **Network issue** → check firewall, DNS, routing

Always start with logs. Logs tell you everything.

```bash
# Your best friend:
docker logs -f <container>  # Follow logs in real-time
```

**Prompt 5 COMPLETE — ALL 8 CAPSTONE MODULES FINISHED**

You now understand this platform from architecture to deployment to debugging.

From here: Run locally, deploy to Hetzner, scale on AWS, and iterate.

---

## Final Thought

This documentation covers:
- How it works (architecture, flows)
- How to build it (docker-compose)
- How to run it (deployment on Hetzner)
- How to scale it (AWS phases)
- How to fix it (debugging)

The platform you've built is production-grade. You've learned:

✓ Docker containerization (all 8 services)
✓ Linux fundamentals (server hardening, networking)
✓ FastAPI web framework (async, dependencies, validation)
✓ PostgreSQL database (queries, migrations, transactions)
✓ Redis caching (sessions, TTLs, queues)
✓ Celery async tasks (workers, beat, retries)
✓ Networking (DNS, TLS, load balancing)
✓ Security (JWT, HTTPS, HMAC, rate limiting)
✓ Distributed systems (eventual consistency, CAP theorem)
✓ AWS managed services (EC2, RDS, ECS, ALB, auto-scaling)

And the capstone: How they all fit together.

Now go build.
