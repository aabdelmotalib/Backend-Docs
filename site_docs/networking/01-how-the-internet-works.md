# Module 1: How the Internet Works

## The Analogy: Postal System

You want to send a letter:
- **Address**: 123 Main Street, New York, NY 10001 — the postal service needs this
- **Port** (apartment number): 5432 — which apartment at that address
- **Protocol** (mail vs package): TCP (guaranteed delivery) vs UDP (fastest path, maybe lost)
- **TCP handshake** (3-way): "Do you accept mail?" "Yes" "Here's my letter" → connection established
- **UDP** (no handshake): "Here's a package!" (hope it arrives, no guarantee)

The internet works similarly. Let's break it down.

## IP Addresses: The Identity Card

Every device on the internet has an address.

### IPv4: The Current Standard

32 bits, written as four decimal numbers: `203.0.113.45`

```
203.0.113.45 breakdown:
203  = first octet (0-255)
0    = second octet
113  = third octet
45   = fourth octet
```

### IPv6: The Future

128 bits, written in hexadecimal: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`

Most of the internet still uses IPv4. IPv6 is newer (1999!) but adoption is slow.

### Public vs Private IP Addresses

**Public**: Routable on the internet
- `8.8.8.8` (Google DNS)
- Your server's IP: `203.0.113.45`

**Private**: Only work inside your network
- `192.168.0.0` to `192.168.255.255`
- `10.0.0.0` to `10.255.255.255`
- `172.16.0.0` to `172.31.255.255`

Your home router: `192.168.1.1` (private). Your laptop gets `192.168.1.100` (private). The router has a public IP that the ISP assigned.

Docker networks work the same way:
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15
    # Gets IP like 172.18.0.2 inside the Docker network
    # Accessible as "postgres" inside the network
  
  api:
    image: my-api
    # Can reach postgres via hostname "postgres" (Docker's DNS)
    # Cannot reach postgres from outside Docker network
```

## Ports: The Mailbox Number

A port is a number 0-65535. It identifies which service on the machine should receive the message.

### Common Ports

```
Port 22:   SSH (secure shell)
Port 80:   HTTP (unencrypted web)
Port 443:  HTTPS (encrypted web)
Port 3306: MySQL database
Port 5432: PostgreSQL database
Port 6379: Redis cache
Port 9000: MinIO S3 storage
Port 8080: Common alternative HTTP
Port 5555: Flower (Celery monitoring)
```

Why these specific numbers? Historically assigned. For internal use, pick anything 1024+.

### The Privilege Rule

Ports 0-1023 require root/administrator. Ports 1024+ are unprivileged. This is why:
- Nginx/HAProxy (reverse proxies needing port 80/443) run as root or in special containers
- Your FastAPI runs on port 8000 (unprivileged)
- Nginx proxies external port 80 to internal port 8000

## TCP vs UDP: Reliable vs Fast

### TCP (Transmission Control Protocol)

**Reliable delivery**, in order, but slower.

Process:
1. SYN: "Can we connect?"
2. SYN-ACK: "Yes"
3. ACK: "OK, session established"
4. (Data transfer)
5. FIN: "Closing"

This 3-way handshake adds latency but guarantees delivery.

Used for:
- HTTP/HTTPS (web)
- SSH (remote access)
- Database connections (PostgreSQL)
- File transfers (very important data)

### UDP (User Datagram Protocol)

**Fast delivery**, no setup, but no guarantee.

Sends immediately: "Here's a packet" (no handshake).

Used for:
- DNS (okay if one packet is lost, try again)
- Voice/video streaming (losing one frame is okay)
- Online games (latency matters more than accuracy)
- SNMP monitoring (okay if health check packet is lost)

### When to Use Each

```python
# Use TCP: you care about reliability and order
# HTTP requests
# Database queries
# File uploads

# Use UDP: you care about latency, not perfect accuracy
# Video streaming
# Real-time multiplayer games
# CoAP (IoT devices)
```

For a PDF processing platform: use TCP (HTTP). You cannot lose a file upload request.

## The Request Journey: browser → server → browser

A user types `example.com` into their browser. Here's the exact journey:

### Step 1: DNS Lookup

Browser: "What's the IP of example.com?"
→ DNS resolver (usually ISP's): Checks authoritative DNS server
→ "example.com is 203.0.113.45"

### Step 2: TCP Handshake

Browser (client): "SYN to 203.0.113.45:443" (HTTPS uses 443)
Server: "SYN-ACK"
Browser: "ACK" → Connection established

### Step 3: TLS Handshake (if HTTPS)

Browser: "What's your certificate?"
Server: "Here's my cert, signed by Let's Encrypt"
Browser: Verifies signature
Both: Agree on encryption key using asymmetric+symmetric crypto
→ Secure connection ready

### Step 4: HTTP Request

Browser sends:
```
GET / HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
...
```

### Step 5: Server Processing

Nginx reverse proxy:
1. Receives encrypted TLS data on port 443
2. Decrypts using certificate's private key
3. Forwards to FastAPI on port 8000 (unencrypted internal network)
4. FastAPI processes request
5. Returns response to Nginx
6. Nginx encrypts with TLS and sends to browser

### Step 6: HTTP Response

Server:
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
...

<html>...</html>
```

Browser renders the page.

## Subnets and CIDR Notation

A subnet is a group of IP addresses.

### CIDR Notation: /XX

`203.0.113.0/24` means:
- Network: `203.0.113.0`
- `/24` = first 24 bits are the network, last 8 bits are hosts
- Addresses: `203.0.113.0` → `203.0.113.255` (256 total, 254 usable)

### Common CIDR Ranges

```
/32 = 1 address (single host)
/30 = 4 addresses (point-to-point link)
/29 = 8 addresses (small subnet)
/27 = 32 addresses
/26 = 64 addresses
/24 = 256 addresses (most common for LANs)
/22 = 1024 addresses
/16 = 65536 addresses
/8 = 16 million addresses
```

Your home router gives your devices:
- Router: `192.168.1.1` (/24 subnet)
- Your PC: `192.168.1.100`
- Phone: `192.168.1.101`

All in the `192.168.1.0/24` subnet.

## Docker's Internal Network

```yaml
version: '3'
services:
  postgres:
    image: postgres:15
    networks:
      - backend

  api:
    image: my-api:latest
    networks:
      - backend

networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/24
```

Docker creates its own subnet `172.18.0.0/24`. Inside this network:
- `postgres` container gets assigned `172.18.0.2`
- `api` container gets assigned `172.18.0.3`
- Docker's DNS resolves `postgres` hostname to `172.18.0.2`

From inside the `api` container:
- `psycopg2.connect("postgres://postgres:5432/mydb")` works ✅
- From outside Docker: `postgres` hostname does not resolve ❌

This isolation is intentional: external traffic cannot directly access `postgres`.

## Hands-On Lab

### Lab 1.1: Trace a Request with traceroute

```bash
# See the path from your computer to Google
traceroute google.com

# Output:
# traceroute to google.com (142.250.185.46), 30 hops max, 60 byte packets
#  1  192.168.1.1 (router)       0.234 ms
#  2  10.123.0.1  (ISP gateway) 12.456 ms
#  3  206.190.36.37 (backbone)   25.123 ms
#  ...
#  10 142.250.185.46 (google.com)  45.678 ms
```

Each line is a "hop" — one router the packet passes through.

### Lab 1.2: Check a Port with ping

```bash
# ICMP echo (does not check specific port)
ping google.com
# PING google.com (142.250.185.46): 56 data bytes
# 64 bytes from 142.250.185.46: icmp_seq=0 ttl=115 time=45.612 ms
```

### Lab 1.3: Check Open Ports with nslookup

```bash
# Get IP address
nslookup google.com
# Name:   google.com
# Address: 142.250.185.46

# Get mail servers for a domain
nslookup -type=MX example.com
```

### Lab 1.4: Docker Network Inspection

```bash
# Start docker-compose
docker-compose up -d

# Inspect network
docker network inspect <network_name>

# Get container IP
docker inspect <container_id> | grep "IPAddress"

# Test DNS resolution
docker exec api_container ping postgres
# ping postgres (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: icmp_seq=0 time=0.123 ms
```

## Cheat Sheet: Networking Commands and Ports

### Common Ports Reference

```
22     SSH (secure shell login)
80     HTTP (unencrypted web)
443    HTTPS (encrypted web)
3306   MySQL
5432   PostgreSQL database
6379   Redis in-memory cache
5672   RabbitMQ message broker
9000   MinIO S3 object storage
8000   Common Python development port
3000   Common Node.js development port
5555   Flower (Celery monitoring)
```

### Core Commands

```bash
# Check DNS resolution
nslookup example.com
dig example.com

# Trace route to a destination
traceroute google.com
mtr google.com  # continuous traceroute

# Check if host is reachable
ping 8.8.8.8

# Show all open ports on your machine
ss -tlnp  # on Linux
netstat -tlnp  # alternative

# Check if a specific port is listening
ss -tlnp | grep 5432

# Resolve hostname to IP
getent hosts postgres  # in Docker
host example.com
```

### CIDR Quick Reference

```
/32 = 1 IP
/30 = 4 IPs
/24 = 256 IPs (most common LAN)
/16 = 65,536 IPs
/8 = 16,777,216 IPs
```

## Key Takeaways

- **IP addresses** identify devices; ports identify services on those devices
- **TCP = reliable** (handshake, guaranteed delivery); UDP = fast (no handshake, no guarantee)
- **Every request travels multiple hops** through routers; traceroute shows the path
- **Subnets** group IP addresses; Docker uses its own internal subnets
- **DNS resolves domains to IPs**; this is the first step in every request
- **Firewalls block ports**; only expose what's necessary

Module 2 teaches HTTP and HTTPS — the protocols that run on top of these network foundations.
