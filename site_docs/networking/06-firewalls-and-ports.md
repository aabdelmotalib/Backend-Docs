# Module 6: Firewalls and Ports

## The Analogy: The Border Gate

A firewall is like a border gate:
- Who can pass? Only those with correct credentials
- Which door can they use? Only authorized entrances
- What weapons are they carrying? (ports carrying unexpected traffic)

A server with no firewall is like a city with no locks on the doors.

## Why Firewalls Matter

Your infrastructure:
```
┌────────────────────────────────────────┐
│ Server (Ubuntu 22.04)                  │
├────────────────────────────────────────┤
│ Port 22: SSH (remote login)            │
│ Port 80: HTTP (web traffic)            │
│ Port 443: HTTPS (encrypted web)        │
│ Port 5432: PostgreSQL (database)       │
│ Port 6379: Redis (cache)               │
│ Port 9000: MinIO (object storage)      │
│ Port 5555: Flower (monitoring)         │
│ ... many others                        │
└────────────────────────────────────────┘
```

Without a firewall: All ports accept connections from anywhere on the internet.
- Attacker scans port 5432, finds PostgreSQL, tries to crack the password
- Attacker scans port 6379, finds Redis, executes commands
- Attacker scans port 9000, finds MinIO, steals files

With a firewall: Only ports 22, 80, 443 are open publicly. The rest are inaccessible.

## ufw: Uncomplicated Firewall

Simple firewall tool on Ubuntu.

### Installation

```bash
# Usually pre-installed
sudo apt-get install ufw

# Enable
sudo ufw enable

# Check status
sudo ufw status
```

### Basic Commands

```bash
# Allow a port
sudo ufw allow 22/tcp         # SSH
sudo ufw allow 80/tcp         # HTTP
sudo ufw allow 443/tcp        # HTTPS

# Deny a port
sudo ufw deny 5432/tcp        # Block PostgreSQL

# Delete a rule
sudo ufw status numbered
# Status numbered to see rule IDs
sudo ufw delete <ID>

# Show all rules
sudo ufw show added

# Reset to defaults
sudo ufw reset
```

### Practical Setup

```bash
# Start fresh
sudo ufw reset

# Allow SSH (access from anywhere needed, but you can restrict to your office IP)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS from everywhere
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Deny everything else (default)
sudo ufw default deny incoming

# Enable
sudo ufw enable

# Verify
sudo ufw status
# Status: active
# 
# To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW       Anywhere
# 80/tcp                     ALLOW       Anywhere
# 443/tcp                    ALLOW       Anywhere
```

## Advanced: Restrict SSH to Your IP

Your office IP is `203.0.113.100`.

```bash
# Allow SSH only from your office
sudo ufw allow from 203.0.113.100 to any port 22

# Allow SSH from a subnet
sudo ufw allow from 203.0.113.0/24 to any port 22

# Check the rule
sudo ufw status numbered
# 1    22/tcp    ALLOW FROM 203.0.113.100
```

Benefits:
- Automated bots cannot brute-force SSH
- You're the only one who can SSH

Warning: If you restrict to wrong IP and get locked out, you need console access to fix it.

## Internal Network: Docker Networking

PostgreSQL, Redis, MinIO run inside Docker. They don't need to be exposed to the firewall.

```yaml
version: '3'
services:
  postgres:
    image: postgres:15
    # NOT published to host
    # Only accessible from other containers
    # Not visible to firewall rules

  redis:
    image: redis:7
    # NOT published to host

  api:
    image: my-api
    ports:
      - "8000:8000"  # Published: container 8000 → host 8000
    # Nginx will proxy to this

  nginx:
    image: nginx
    ports:
      - "80:80"      # HTTP
      - "443:443"    # HTTPS
```

Docker creates its own internal network. PostgreSQL is only accessible from inside the Docker network, not from the public internet.

Firewall rules don't apply to Docker's internal network. But you should still restrict externally published ports.

## SSH Hardening: Beyond the Firewall

Firewall blocks unauthorized connections. SSH hardening makes authorized connections secure.

### Disable Password Authentication

Use key-based auth only (more secure):

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes

# Reload SSH config
sudo systemctl reload ssh
```

Benefits:
- Attackers cannot brute-force passwords
- Only your key can access the server

Setup:
```bash
# Generate key pair (on your local machine)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519 user@server.com

# Test
ssh -i ~/.ssh/id_ed25519 user@server.com
```

### Change Default SSH Port

SSH runs on port 22. Attackers scan this port. Change it:

```bash
# /etc/ssh/sshd_config
Port 2222

# Firewall rule
sudo ufw allow 2222/tcp

# Reload SSH
sudo systemctl reload ssh

# Connect
ssh -p 2222 user@server.com
```

Downside: You have to remember the custom port. Less convenient.

## Fail2ban: Automatic IP Banning

If someone tries to SSH with wrong password 5 times in 10 minutes, ban their IP for 1 hour.

### Installation

```bash
sudo apt-get install fail2ban

# Enable
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Configuration

```bash
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
maxretry = 5      # Ban after 5 failed attempts
findtime = 600    # In 10 minutes
bantime = 3600    # Ban for 1 hour

[rate-limit]
enabled = true
port = http,https
maxretry = 100    # Ban after 100 requests
findtime = 60     # In 1 minute
bantime = 3600    # Ban for 1 hour
```

### Monitor Bans

```bash
# See failed login attempts
sudo tail -f /var/log/auth.log | grep "Failed password"

# See bans
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client set sshd unbanip 203.0.113.50
```

## Network Segmentation: Defense in Depth

Multiple layers of security:

```
Layer 1: Firewall (public internet)
  ↓ Only 22, 80, 443 allowed
Layer 2: Reverse Proxy (Nginx)
  ↓ Authenticate, rate limit, validate requests
Layer 3: API (FastAPI)
  ↓ Authorization checks, input validation
Layer 4: Database (PostgreSQL)
  ↓ Only accepts connections from API container
```

Each layer validates. Even if one is compromised, others protect.

## Hands-On Lab

### Lab 6.1: Configure ufw

```bash
# Start fresh
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH, HTTP, HTTPS
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable
sudo ufw enable

# Verify
sudo ufw status
```

### Lab 6.2: Verify PostgreSQL is Blocked

```bash
# From outside the server, try to connect
psql -h server-ip -U postgres

# Should time out or refuse (port 5432 is not open)
# ❌ connection refused

# From inside the server (or Docker), should work
docker exec postgres psql -U postgres -c "SELECT 1"
# ✅ works
```

### Lab 6.3: Restrict SSH to Your IP

Find your public IP:
```bash
curl https://ifconfig.me
# 203.0.113.100
```

Then:
```bash
# Remove the old rule
sudo ufw status numbered
sudo ufw delete 1  # Delete rule 1 (old SSH rule)

# Add new rule
sudo ufw allow from 203.0.113.100 to any port 22

# Verify
sudo ufw status
```

## Cheat Sheet: Firewall and Security

### ufw Quick Reference

```bash
sudo ufw enable                      # Enable firewall
sudo ufw status                      # Show rules
sudo ufw allow 22/tcp                # Allow SSH
sudo ufw deny 5432/tcp               # Block PostgreSQL
sudo ufw allow from IP to any port X # Allow from specific IP
sudo ufw reset                       # Clear all rules
```

### SSH Hardening

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no             # Disable password auth
PubkeyAuthentication yes              # Key-based only
Port 2222                             # Custom port
PermitRootLogin no                    # Never login as root
```

### Fail2ban

```bash
sudo fail2ban-client status sshd      # View bans
sudo fail2ban-client set sshd unbanip IP  # Unban IP
```

## Security Checklist

```
Network Layer
☐ Firewall enabled (ufw enable)
☐ Only ports 22, 80, 443 open
☐ All other ports denied
☐ PostgreSQL, Redis, MinIO not exposed

SSH Layer
☐ Key-based authentication only (PasswordAuthentication no)
☐ Root login disabled (PermitRootLogin no)
☐ Non-standard port (Port 2222)
☐ Fail2ban running (auto-ban after failed attempts)

API Layer
☐ All API endpoints authenticate users
☐ Rate limiting enabled
☐ Input validation on all endpoints

Database Layer
☐ PostgreSQL only accepts connections from API container
☐ Strong password (50+ characters)
☐ No public IP exposure
```

## Key Takeaways

- **Firewall = first defense** — only expose what's necessary
- **ufw = simple firewall tool** — allow/deny ports easily
- **Only open 22 (SSH), 80 (HTTP), 443 (HTTPS)** — everything else is blocked
- **SSH hardening** = key-based auth, disable passwords, custom port
- **Fail2ban = automatic protection** against brute-force attacks
- **Docker internal network** = databases don't need firewall rules
- **Defense in depth** = multiple security layers (firewall, proxy, auth, database)

Networking section complete. Next: Security fundamentals.
