# Module 3: DNS Explained

## The Analogy: The Phone Book

Your friend: "Call Alice!"
You: "I don't know her number. Let me check the phone book."
Phone book: "Alice is 555-1234"
You: "Thanks! Now I can call 555-1234"

DNS works the same way:

Browser: "I want to visit example.com"
DNS resolver: "Let me check..."
DNS: "example.com is 203.0.113.45"
Browser: "Thanks! Now I can connect to 203.0.113.45"

Without DNS, you'd have to type `203.0.113.45` in your browser. Unworkable. DNS makes the internet human-friendly.

## What DNS Does

Translates domain names (hostnames) to IP addresses.

```
example.com         → 203.0.113.45
api.example.com     → 203.0.113.46
mail.example.com    → 203.0.113.47
```

## DNS Records: Different Types

When you own a domain, you configure records that tell DNS what to return.

### A Record: IPv4

```
example.com  A  203.0.113.45
```

"The IPv4 address for example.com is 203.0.113.45"

Used for: pointing your domain to your server

### AAAA Record: IPv6

```
example.com  AAAA  2001:db8::1
```

"The IPv6 address for example.com is 2001:db8::1"

Most servers only have IPv4. IPv6 adoption is slow.

### CNAME Record: Alias

```
www.example.com  CNAME  example.com
```

"www.example.com is an alias for example.com"

Useful:
```
api.example.com         CNAME  example.com
subdomain.example.com   CNAME  example.com
cdn.example.com         CNAME  cloudflare.example.com  # CDN
```

Don't put a CNAME on the root domain (`example.com`). Use A record for the root, CNAME for subdomains.

### MX Record: Mail Exchange

```
example.com  MX  10 mail.example.com
example.com  MX  20 mail2.example.com
```

"Mail for example.com goes to mail.example.com (priority 10) or mail2.example.com (priority 20)"

The number is priority. Lower number is tried first (failover).

Used for: email routing (not for HTTP APIs)

### TXT Record: Text Data

```
example.com  TXT  "v=spf1 include:google.com ~all"
```

Arbitrary text. Commonly used for:
- **SPF**: Email authentication
- **DKIM**: Email signing
- **DMARC**: Email policy
- **Let's Encrypt verification**: `_acme-challenge.example.com  TXT  validation_token_12345`

### NS Record: Name Server

Points to the servers that store your DNS records.

```
example.com  NS  ns1.example.com
example.com  NS  ns2.example.com
```

Your registrar (GoDaddy, Namecheap) has default NS records. If you use Route53 or Cloudflare, they provide their own NS records.

## DNS Propagation: The 48-Hour Wait

When you change a DNS record, it does not change everywhere instantly.

### Timeline

You: "Change example.com A record to 203.0.113.50"
Your registrar: "Done"
Your local ISP's DNS cache: Might still have the old record (cached)
Other countries' DNS: Might take 24-48 hours to get the update

In practice:
- 5-30 minutes: Most of the world
- 24 hours: Your local ISP
- 48 hours: Everywhere (guaranteed by DNS standard)

During this time, some users see the old IP, some see the new IP.

### Workaround: Lower TTL Before Migration

TTL = Time To Live = How long DNS servers cache the record.

```
example.com  A  203.0.113.45  TTL 86400  (1 day)
```

High TTL (86400 = 1 day): Faster lookups, but slower to update
Low TTL (300 = 5 minutes): Can update quickly, but more DNS queries

Before migrating:
1. Lower TTL to 300 (24 hours before)
2. Make your DNS changes
3. Wait for propagation
4. Raise TTL back to 86400 after migration

## DNS in Docker

Docker has built-in DNS so containers can find each other by hostname.

```yaml
version: '3'
services:
  postgres:
    image: postgres:15
    container_name: postgres

  api:
    image: my-api
    container_name: api
    depends_on:
      - postgres
```

From inside the `api` container:
- Cannot use `localhost:5432` (that's the API container itself)
- Use `postgres:5432` (Docker's DNS resolves "postgres" to the postgres container's IP)

How it works:
1. Docker's embedded DNS server responds to `.queries inside the network
2. `postgres` resolves to the postgres container's IP (e.g., 172.18.0.2)
3. API connects to `172.18.0.2:5432`

## Let's Encrypt DNS Challenge

When you get a free certificate from Let's Encrypt, they verify you own the domain:

### HTTP Challenge (simple)

Let's Encrypt: "Prove you own example.com"
You: "I'll put this token on my server"
You: "Generate file at `/.well-known/acme-challenge/token123`"
(Certbot does this automatically)
Let's Encrypt: Checks `example.com/.well-known/acme-challenge/token123`
Let's Encrypt: "Verified! Here's your certificate"

### DNS Challenge (for wildcards and complex setups)

Let's Encrypt: "Prove you own example.com"
You: "I'll add this to DNS"
You: `_acme-challenge.example.com  TXT  validation_token_abc123`
Let's Encrypt: Checks DNS
Let's Encrypt: "Verified! Here's your certificate"

Used for wildcard certificates:
```
*.example.com  (covers api.example.com, mail.example.com, etc.)
```

## Hands-On Lab

### Lab 3.1: Resolve a Domain with dig and nslookup

```bash
# Using dig (more detailed)
dig example.com
# ANSWER SECTION:
# example.com.  300  IN  A  203.0.113.45

# Using nslookup (simpler)
nslookup example.com
# Name:   example.com
# Address: 203.0.113.45

# Get a specific record type
dig example.com MX
# Shows mail servers

dig example.com TXT
# Shows text records
```

### Lab 3.2: Find DNS Servers for a Domain

```bash
# Which servers are authoritative for this domain?
dig example.com NS
# example.com.  300  IN  NS  ns1.example.com
# example.com.  300  IN  NS  ns2.example.com

# Get the NS servers' IP addresses
dig @ns1.example.com example.com A
```

### Lab 3.3: Check Propagation

```bash
# Check DNS record from multiple resolvers
dig @8.8.8.8 example.com       # Google's DNS
dig @1.1.1.1 example.com       # Cloudflare's DNS
dig @208.67.222.222 example.com # OpenDNS

# All should return same IP if propagated
```

### Lab 3.4: Add A Record to Your Domain

In your registrar (GoDaddy, Namecheap, Route53):
1. Add A record: `example.com → YOUR_SERVER_IP`
2. Wait for propagation (usually 5-30 minutes)
3. Verify: `dig example.com` shows your IP
4. Try to SSH: `ssh user@example.com` (should work if SSH is running)

## Cheat Sheet: DNS

### Record Types

```
A       = Domain → IPv4 (most common)
AAAA    = Domain → IPv6
CNAME   = Alias to another domain
MX      = Mail server for this domain
NS      = Nameserver (points to DNS provider)
TXT     = Arbitrary text (verification, signing)
SOA     = Start of Authority (zone info)
```

### Common dig/nslookup Commands

```bash
dig example.com              # Look up A record
dig example.com A            # Explicitly ask for A record
dig example.com AAAA         # Look up IPv6 (AAAA)
dig example.com MX           # Mail servers
dig example.com NS           # Authoritative nameservers
dig example.com TXT          # Text records
dig @8.8.8.8 example.com     # Query specific DNS server

nslookup example.com         # Simple lookup
nslookup example.com 8.8.8.8 # Query specific server
```

### TTL Interpretation

```
300 = 5 minutes (fast updates)
3600 = 1 hour (medium)
86400 = 24 hours (standard)
604800 = 7 days (long, rarely used)
```

### Propagation Debugging

```bash
# Check what your registrar says
# (usually available in registrar's web interface)

# Check what resolvers see globally
dig @8.8.8.8 example.com     # Google
dig @1.1.1.1 example.com     # Cloudflare

# If different, not yet propagated
# Wait and retry
```

## Key Takeaways

- **DNS translates domains to IPs** — human names to machine addresses
- **A record** points domain to IPv4 (most common)
- **CNAME** creates aliases
- **TTL** controls how long to cache
- **Propagation takes up to 48 hours** — lower TTL before migrations
- **Lower TTL before changes** to speed up propagation
- **Docker DNS** lets containers find each other by hostname
- **Let's Encrypt** verifies domain ownership via HTTP or DNS challenge

Module 4 teaches Nginx — the reverse proxy that sits in front of your API.
