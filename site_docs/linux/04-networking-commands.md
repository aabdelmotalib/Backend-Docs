# Module 4: Networking Commands

## The Analogy: The Postal System

The internet is like the postal system:

- **Your computer** is your house with an address (IP address)
- **Ports** are specific rooms in your house (port 80 for HTTP, port 3306 for MySQL)
- **DNS** is the phonebook translating names (google.com) to addresses (142.250.80.46)
- **curl** is you sending a letter with requests and receiving responses

Understanding networking means understanding how data travels between computers.

## Essential Networking Concepts

### IP Address

Every computer on a network has an IP address:

- **IPv4**: `192.168.1.1` (4 numbers, each 0-255)
- **IPv6**: `2001:0db8:85a3::8a2e:0370:7334` (longer, newer standard)

Public vs Private:

- **Public**: Accessible from the internet (your server's public IP)
- **Private**: Only accessible on local network (192.168.0.0/16, 10.0.0.0/8)

### Ports

Ports are numbered channels (0-65535) for different services:

- **Port 80**: HTTP (web)
- **Port 443**: HTTPS (secure web)
- **Port 22**: SSH (remote access)
- **Port 5432**: PostgreSQL
- **Port 3306**: MySQL
- **Port 6379**: Redis
- **Port 8000**: Often used for development APIs

### DNS

Translates human-readable names to IP addresses:

- `google.com` → `142.250.80.46`
- `api.example.com` → `13.225.1.10`

## Viewing Network Interfaces: ip addr and ifconfig

### ip addr — View Network Interfaces

```bash
ip addr
# or
ip addr show
```

Output:

```
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8
    inet6 ::1/128

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.100/24
    inet6 fe80::1/64
```

Explanation:
- `lo` = loopback (localhost, 127.0.0.1)
- `eth0` = ethernet interface (your network card)
- `192.168.1.100` = your IP address
- `/24` = subnet mask

### ifconfig — Legacy Alternative

Older systems use `ifconfig` (deprecated but still works):

```bash
ifconfig
```

## curl — The Most Important Tool

**curl** is the Swiss Army knife for API testing. You'll use it constantly.

### Basic GET Request

```bash
curl https://api.example.com/users
```

Response is printed to stdout.

### GET with Headers

```bash
curl -H "Authorization: Bearer token123" https://api.example.com/users
curl -H "Accept: application/json" https://api.example.com/users
curl -H "User-Agent: MyApp/1.0" https://api.example.com/users
```

### POST Request with Data

```bash
curl -X POST -d "name=Alice&email=alice@example.com" https://api.example.com/users
```

### POST with JSON

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}' \
  https://api.example.com/users
```

### Authentication

```bash
# Basic auth (username:password in header)
curl -u username:password https://api.example.com/secure

# Bearer token
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.example.com/secure

# API key
curl -H "X-API-Key: your-api-key" https://api.example.com/secure
```

### File Upload

```bash
curl -F "file=@/path/to/file.txt" https://api.example.com/upload
```

The `-F` flag sends the file as multipart form data.

### Useful Flags

```bash
curl -v https://api.example.com          # Verbose (show headers)
curl -i https://api.example.com          # Include response headers
curl -o output.html https://example.com  # Save to file
curl -L https://example.com              # Follow redirects
curl -H "Host: example.com" http://1.2.3.4  # Override Host header
```

## wget — Download Files

```bash
wget https://example.com/file.tar.gz
wget -O custom_name.tar.gz https://example.com/file.tar.gz
```

## ping — Test Connectivity

```bash
ping google.com
```

Output:

```
PING google.com (142.250.80.46) 56(84) bytes of data.
64 bytes from 142.250.80.46: icmp_seq=1 ttl=56 time=15.2 ms
```

Means: Successfully reached google.com, response time 15.2ms.

```bash
ping -c 5 google.com    # Send 5 pings and stop
```

## traceroute — Path to a Host

Shows every hop (router) between you and a destination:

```bash
traceroute google.com
```

Output:

```
 1    192.168.1.1 (your gateway)
 2    10.0.0.1 (ISP)
 3    203.0.113.5 (Internet backbone)
 ...
10    142.250.80.46 (google.com)
```

## nslookup and dig — DNS Queries

### nslookup

```bash
nslookup google.com
# Output:
# Non-authoritative answer:
# Name:   google.com
# Address: 142.250.80.46
```

### dig

More detailed DNS information:

```bash
dig google.com
# Shows MX records, NS records, full DNS data
```

## netstat and ss — Open Ports and Connections

### netstat — Network Statistics

```bash
netstat -tlnp        # -t (TCP), -l (listening), -n (numeric), -p (program)
```

Output:

```
Proto Recv-Q Send-Q Local Address           State       PID/Program
tcp   0      0      127.0.0.1:5432          LISTEN      1234/postgres
tcp   0      0      0.0.0.0:8000            LISTEN      5678/python
tcp   0      0      192.168.1.1:22          LISTEN      9012/sshd
```

Shows services listening on ports.

### ss — Socket Statistics (Modern)

```bash
ss -tlnp      # Same flags
```

More modern and faster than netstat.

## ufw — Simple Firewall

Ubuntu's simple firewall:

```bash
sudo ufw status                     # Check status
sudo ufw enable                     # Enable firewall
sudo ufw disable                    # Disable firewall
sudo ufw allow 22/tcp               # Allow SSH
sudo ufw allow 80/tcp               # Allow HTTP
sudo ufw allow 443/tcp              # Allow HTTPS
sudo ufw deny 3306/tcp              # Deny MySQL
sudo ufw delete allow 3306/tcp      # Remove rule
```

## Hands-On Lab

### Lab 4.1: Check Your Network Configuration

```bash
# View your IP address
ip addr

# View all interfaces
ip link show

# Check connectivity to Google
ping -c 3 google.com

# Trace the path to Google
traceroute google.com
```

### Lab 4.2: DNS Queries

```bash
# Resolve domain to IP
nslookup google.com

# Detailed DNS info
dig google.com

# Query specific DNS record
dig google.com A      # IPv4 address
dig google.com MX     # Mail server
```

### Lab 4.3: Testing HTTP APIs with curl

#### GET Request

```bash
curl https://jsonplaceholder.typicode.com/posts/1
```

(This is a fake API for testing)

#### POST Request

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"title":"My Post","body":"Content"}' \
  https://jsonplaceholder.typicode.com/posts
```

#### With Headers

```bash
curl -v https://jsonplaceholder.typicode.com/posts/1
# -v shows request/response headers
```

#### Custom Headers

```bash
curl -H "User-Agent: MyBot/1.0" https://jsonplaceholder.typicode.com/posts/1
```

#### Basic Authentication

```bash
curl -u username:password https://api.example.com/protected
```

### Lab 4.4: Check Open Ports

```bash
# List all listening ports (requires root)
sudo ss -tlnp

# Or use netstat
sudo netstat -tlnp

# Check if a specific port is listening
ss -tlnp | grep 8000  # Checking port 8000
```

### Lab 4.5: Test Your Local API

If you have a running API (FastAPI, Django, etc.) on localhost:8000:

```bash
# Simple GET
curl http://localhost:8000/

# GET with path
curl http://localhost:8000/health

# POST with JSON
curl -X POST -H "Content-Type: application/json" \
  -d '{"name":"Test"}' \
  http://localhost:8000/items

# With verbose
curl -v http://localhost:8000/
```

## Cheat Sheet: curl for API Testing

### HTTP Methods

```bash
curl http://api.example.com/item              # GET (default)
curl -X GET http://api.example.com/item       # Explicit GET
curl -X POST http://api.example.com/items     # POST
curl -X PUT http://api.example.com/item/1    # PUT
curl -X PATCH http://api.example.com/item/1  # PATCH
curl -X DELETE http://api.example.com/item/1 # DELETE
```

### Headers

```bash
curl -H "Content-Type: application/json" http://api.example.com
curl -H "Authorization: Bearer TOKEN" http://api.example.com
curl -H "Custom-Header: value" http://api.example.com
curl -d @payload.json http://api.example.com  # Send file as body
```

### Response Handling

```bash
curl http://api.example.com -o response.json  # Save to file
curl http://api.example.com | jq              # Pipe to jq (JSON parser)
curl -i http://api.example.com                # Include headers
curl -I http://api.example.com                # Headers only
curl -w "%{http_code}\n" http://api.example.com  # Just HTTP status
```

### Common Patterns

```bash
# API test with JSON
curl -X POST -H "Content-Type: application/json" \
  -d '{"key":"value"}' \
  http://api.example.com/endpoint

# File upload
curl -F "file=@path/to/file" http://api.example.com/upload

# Basic auth
curl -u user:pass http://api.example.com

# Custom timeout
curl --max-time 5 http://api.example.com

# Follow redirects
curl -L http://short.link.com
```

## Key Takeaways

- **ip addr** shows your network configuration
- **ping** tests if a host is reachable
- **curl** is essential for API testing (GET, POST, headers, auth)
- **nslookup/dig** translates domain names to IPs
- **ss -tlnp** shows what services are listening
- **ufw** provides simple firewall rules
- **curl with -H** adds custom headers (auth, content-type)
- **curl with -d** sends data (JSON, form fields)

Module 5, the final module, teaches you to write bash scripts that automate all of this.
