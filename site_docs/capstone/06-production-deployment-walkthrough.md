# Module 6: Production Deployment Walkthrough

## From Code to Live Internet: Step by Step

You've built the platform locally. Now deploy it to the internet so real users can access it.

Target: Hetzner CX21 VPS in Frankfurt, EU-WEST-1.

---

## Step 1: Rent Hetzner VPS

Visit hetzner.com:

1. Click "Cloud" → "Servers"
2. Choose **CX21**:
   - 4 vCPU
   - 8GB RAM
   - 160GB NVMe SSD
   - **€8.50/month** (best value)
3. Operating System: **Ubuntu 22.04 LTS** (long-term support)
4. Location: **Falkenstein DC8** (Frankfurt region)
5. SSH key: Generate and save locally (or Hetzner generates, download)

Order created. Within 30 seconds: Server is live.

Get server IP: `203.0.113.77` (example)

---

## Step 2: Server Hardening

SSH into the server:

```bash
ssh root@203.0.113.77
# First login: root user

# Change hostname
hostnamectl set-hostname pdf-platform

# Update system
apt update && apt upgrade -y

# Set timezone
timedatectl set-timezone Europe/Berlin
```

### Firewall (ufw)

```bash
# Enable firewall
ufw enable

# Allow SSH (critical! don't lock yourself out)
ufw allow 22/tcp

# Allow HTTP/HTTPS
ufw allow 80/tcp
ufw allow 443/tcp

# Deny everything else
ufw default deny incoming
ufw default allow outgoing

# Verify rules
ufw status
```

### Fail2Ban (Brute-Force Protection)

```bash
# Install
apt install fail2ban -y

# Auto-configure
systemctl enable fail2ban
systemctl start fail2ban

# Monitor
fail2ban-client status sshd
```

---

## Step 3: Install Docker

```bash
# Add Docker repo (from Module 1, Docker & Containers)
apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install
apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

# Start daemon
systemctl enable docker
systemctl start docker

# Test
docker --version
docker run hello-world
```

---

## Step 4: DNS Configuration

You own domain: `pdf-platform.example.com`

At your DNS provider (Namecheap, GoDaddy, etc.):

Create **A record**:
```
Name:  @
Type:  A
Value: 203.0.113.77  (your Hetzner server IP)
TTL:   3600
```

Also create **www** subdomain:
```
Name:  www
Type:  A
Value: 203.0.113.77
TTL:   3600
```

Save. **Queue for propagation** (5-30 minutes globally).

Verify propagation:
```bash
dig @8.8.8.8 pdf-platform.example.com
# Should return: 203.0.113.77
```

---

## Step 5: Clone Repository

On server:

```bash
# Install git
apt install git -y

# Create app directory
mkdir -p /opt/pdf-platform
cd /opt/pdf-platform

# Clone your repo
git clone https://github.com/your-username/pdf-platform.git .

# Verify structure
ls -la
# Should show: docker-compose.prod.yml, .env.example, etc.
```

---

## Step 6: Create Production .env

```bash
# Copy template
cp .env.example .env.prod

# Edit with production values
nano .env.prod
```

Contents:

```bash
# === Environment ===
ENVIRONMENT=production
DEBUG=False

# === Database ===
DATABASE_URL=postgresql://pdf_user:RANDOM_PASSWORD@postgresql:5432/pdf_db
DATABASE_POOL_SIZE=20

# === Redis ===
REDIS_URL=redis://redis:6379

# === JWT & Security ===
SECRET_KEY=$(openssl rand -hex 32)  # Generate random key!
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
JWT_SECRET_KEY=$(openssl rand -hex 32)

# === MinIO ===
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=$(openssl rand -hex 16)
MINIO_ENDPOINT=http://minio:9000
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=$(value above)

# === ClamAV ===
CLAMAV_SOCKET=clamav:3310

# === Paymob ===
PAYMOB_API_KEY=your_actual_api_key_here
PAYMOB_INTEGR_ID=123456  # Your integration ID

# === Email ===
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password  # Generate from Gmail settings
ADMIN_EMAIL=admin@example.com

# === Celery ===
CELERY_BROKER_URL=redis://redis:6379
CELERY_RESULT_BACKEND=redis://redis:6379

# === Logging ===
LOG_LEVEL=info
SENTRY_DSN=https://your-sentry-dsn@sentry.io/123456  # Optional

# === Frontend ===
REACT_APP_API_URL=https://pdf-platform.example.com
REACT_APP_ENV=production
```

Generate random keys:
```bash
# In your terminal
openssl rand -hex 32  # For SECRET_KEY
openssl rand -hex 16  # For password
```

---

## Step 7: Start Docker Compose

```bash
# Make sure .env.prod exists
ls -la .env.prod

# Start all services in background
docker compose -f docker-compose.prod.yml up -d

# Watch logs
docker compose -f docker-compose.prod.yml logs -f api

# Check all services
docker compose -f docker-compose.prod.yml ps
# Should see: api (running), postgresql (running), redis (running), etc.

# Wait 30 seconds for services to initialize
# Then verify health
curl -s http://localhost:8000/health | jq .
```

---

## Step 8: SSL Certificate (Let's Encrypt)

HTTPS is mandatory (users trust, browser padlock).

```bash
# Install Certbot
apt install certbot python3-certbot-nginx -y

# Request certificate (for your domain)
certbot --nginx -d pdf-platform.example.com -d www.pdf-platform.example.com

# Interactive prompts:
#   Email: your-email@example.com
#   Agree to terms: Yes (A)
#   Subscribe to newsletter: No (N)

# Certbot auto-configures Nginx!
# Automatically enables HTTPS redirect (80 → 443)
```

Test certificate:
```bash
curl -I https://pdf-platform.example.com
# Should show: 200 OK, lock icon in browser
```

---

## Step 9: Verify HTTPS Works

```bash
# In browser
https://pdf-platform.example.com

# Should show:
# ✓ Padlock icon (secure)
# ✓ Green "Secure" label
# ✓ Certificate: Let's Encrypt
```

View certificate details:
```bash
# From server
certbot certificates

# Check expry
openssl s_client -connect pdf-platform.example.com:443 2>/dev/null | grep "Not After"
```

---

## Step 10: Update Paymob Webhook URL

In Paymob dashboard settings:

Change webhook URL from:
```
http://localhost:8000/payments/webhook
```

To live domain:
```
https://pdf-platform.example.com/payments/webhook
```

Test webhook:
```bash
# Paymob dashboard → Settings → Webhooks → Test

# Check server received it
docker logs api | grep "webhook"
```

---

## Step 11: Monitoring (UptimeRobot)

UptimeRobot monitors your site, alerts if down.

Visit uptimerobot.com:

1. Sign up free
2. New monitor:
   - URL: `https://pdf-platform.example.com/health`
   - Type: HTTPS
   - Interval: 5 minutes
   - Uptime password: (none)
3. Alert destination: Your email

Now: If site goes down, you get alerted within 5 minutes.

---

## Step 12: Automatic Backups

PostgreSQL and MinIO data must be backed up.

### Option A: Hetzner Snapshots

Hetzner can create snapshots of your VPS disk:

```bash
# Manual snapshot (one-time)
hcloud server create-image --type backup pdf-platform

# Or in Hetzner UI
# Server dashboard → Snapshots → Create
```

### Option B: Database Backup Script

```bash
#!/bin/bash
# /opt/pdf-platform/backup.sh

BACKUP_DIR="/opt/backups"
TIMESTAMP=$(date +%Y-%m-%d-%H%M%S)

# Backup PostgreSQL
docker exec postgresql pg_dump -U pdf_user pdf_db | gzip > $BACKUP_DIR/db-$TIMESTAMP.sql.gz

# Backup MinIO files
docker exec minio mc mirror /minio/uploads s3://backups/$TIMESTAMP/uploads

# Keep only last 30 days
find $BACKUP_DIR -mtime +30 -delete

# Upload to external storage
aws s3 cp $BACKUP_DIR/ s3://my-backup-bucket/ --recursive

echo "Backup completed: $TIMESTAMP"
```

Make executable:
```bash
chmod +x /opt/pdf-platform/backup.sh
```

Schedule with cron (daily at 2 AM):
```bash
crontab -e

# Add line:
0 2 * * * /opt/pdf-platform/backup.sh
```

---

## Runbook: When Site Goes Down

You get alert at 3 AM. Site is offline. What now?

### First: Confirm It's Really Down

```bash
curl -I https://pdf-platform.example.com
# Should return 200, if not...

curl -I http://localhost:8000/health
# Check Nginx is up

# Check Docker services
docker compose -f docker-compose.prod.yml ps

# Are containers running?
# CONTAINER            STATUS
# api                  Up 48 hours
# postgresql           Up 48 hours
# redis                Up 48 hours
# etc.
```

### Common Scenarios

**Scenario 1: Container Crashed**

```bash
# Check logs
docker logs api | tail -50

# Restart
docker restart api

# Verify
curl http://localhost:8000/health
```

**Scenario 2: Out of Disk Space**

```bash
# Check
df -h

# MinIO data too big?
du -sh /var/lib/docker/volumes/pdf_minio_data

# Solution:
# - Delete old files from MinIO
# - Or add another disk (Hetzner)
```

**Scenario 3: Database Connection Pool Exhausted**

```bash
# Symptoms: 503 "too many connections"

# Check
docker logs postgresql | grep "too many"

# Increase pool
nano .env.prod
# Change: DATABASE_POOL_SIZE=30

# Restart
docker compose down
docker compose up -d
```

**Scenario 4: SSL Certificate Expired**

```bash
# Check
certbot certificates
# Shows certificate expiry date

# Renewal (automatic, but manual if needed)
certbot renew --force-renewal

# Restart Nginx
docker exec nginx nginx -s reload
```

---

## Production Maintenance: Cheat Sheet

```bash
# View logs (follow mode)
docker compose -f docker-compose.prod.yml logs -f api

# Restart service
docker restart api

# Stop all
docker compose down

# Start all
docker compose -f docker-compose.prod.yml up -d

# Database migrations
docker exec api alembic upgrade head

# Check service health
curl http://localhost:8000/health | jq .

# Check database
docker exec postgresql psql -U pdf_user -d pdf_db -c "SELECT COUNT(*) FROM users;"

# Check Redis
docker exec redis redis-cli PING

# Check MinIO
docker exec minio mc ls minio/uploads

# View environment
cat .env.prod

# Update code (without stopping)
cd /opt/pdf-platform
git pull
docker build -t pdf-api:latest .
docker compose up -d api  # Restart only API

# Scale workers
docker compose -f docker-compose.prod.yml up -d --scale worker=3
# Now 3 Celery workers instead of 1

# Backup manually
docker exec postgresql pg_dump -U pdf_user pdf_db | gzip > backup-$(date +%Y%m%d).sql.gz

# Restore backup
gunzip < backup-20240115.sql.gz | docker exec -i postgresql psql -U pdf_user -d pdf_db

# SSH into server
ssh -i ~/.ssh/id_rsa root@203.0.113.77

# Kill hung task
docker exec api ps aux | grep python
docker exec api kill -9 12345  # PID from above
```

---

## Key Points

- **Hetzner CX21**: €8.50/month for entire platform
- **DNS**: A record points to server IP
- **SSL**: Let's Encrypt (free, auto-renews)
- **Firewall**: Only ports 22 (SSH), 80 (HTTP), 443 (HTTPS)
- **Backups**: Daily snapshots + off-site storage
- **Monitoring**: UptimeRobot alerts if down
- **Logs**: Always `docker logs` to diagnose
- **Scaling**: Add more workers (Celery), more API replicas (load balance)

Next module: Scaling this platform to thousands of users with AWS.
