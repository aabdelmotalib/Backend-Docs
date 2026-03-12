# Module 4: Nginx Reverse Proxy

## The Analogy: The Hotel Concierge

You arrive at a hotel lobby. You ask the concierge:
- "Where's the restaurant?" → Concierge directs you to dining room
- "I need a taxi" → Concierge calls dispatch
- "Room 302, please?" → Concierge sends you upstairs

The concierge is a single point of entry. Guests don't roam the hotel looking for services.

Nginx is a reverse proxy: a single entry point for your backend.

```
Internet traffic (80, 443)  → Nginx (entry point)
                                ↓
                    ┌───────────┼───────────┐
                    ↓           ↓           ↓
                API1      API2         API3
              (8000)    (8000)       (8000)
```

Why have a reverse proxy?

1. **SSL/TLS Termination**: Decrypt HTTPS once, forward unencrypted HTTP internally
2. **Rate Limiting**: Stop malicious requests before they reach the API
3. **Static Files**: Serve React JS/CSS directly without hitting Python
4. **Load Balancing**: Distribute requests across multiple API instances
5. **Compression**: Gzip responses before sending to clients
6. **Caching**: Cache responses to avoid hitting the API repeatedly

## Nginx Configuration Structure

```nginx
http {
  # Global settings for all HTTP traffic
  
  upstream api_backend {
    # Where to forward requests
    server api1:8000;
    server api2:8000;
    server api3:8000;
  }
  
  server {
    # Handle traffic for a specific domain
    listen 80;
    listen 443 ssl;
    server_name example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    location / {
      # Default: proxy to API
      proxy_pass http://api_backend;
    }
    
    location /static/ {
      # Serve React files directly
      alias /app/frontend/build/;
    }
    
    location = /health {
      # Internal health check (no proxying)
      access_log off;
      return 200 "OK";
    }
  }
}
```

## Proxy Pass: Forward Requests

```nginx
location /api/ {
  proxy_pass http://api_backend;
}
```

Client request:
```
GET /api/documents HTTP/1.1
Host: example.com
Authorization: Bearer token
```

Nginx forwards to API:
```
GET /api/documents HTTP/1.1
Host: api1            # Changed to backend
Via: nginx            # Added header (optional)
Authorization: Bearer token  # Pass through
```

API responds, Nginx forwards back to client.

## Rate Limiting

Prevent DDoS and abusive clients.

```nginx
http {
  # Define rate limit zone
  limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
  
  server {
    location /api/ {
      # Apply rate limit: 10 requests per second per IP
      limit_req zone=api_limit burst=20 nodelay;
      
      proxy_pass http://api_backend;
    }
  }
}
```

Explanation:
- `10r/s` = 10 requests per second per IP
- `burst=20` = allow up to 20 in queue before rejecting
- `nodelay` = reject immediately if over limit (vs delay)

Client making 50 requests/second:
- First 10 succeed
- Next 20 are queued (burst)
- Remaining 20 get HTTP 429 (Too Many Requests)

## Load Balancing

Distribute requests across multiple backend servers.

```nginx
upstream api_backend {
  # Round-robin (default): alternate between servers
  server api1:8000;
  server api2:8000;
  server api3:8000;
}

upstream api_backend_weighted {
  # Some servers more powerful
  server api1:8000 weight=5;  # Gets 5x more traffic
  server api2:8000 weight=1;  # Gets normal share
}

upstream api_backend_least_conn {
  # Least connections first (for long-lived requests)
  least_conn;
  server api1:8000;
  server api2:8000;
}
```

## Static Files: Skip the API

Serve React build directly from Nginx:

```nginx
location / {
  # Try static file first
  try_files $uri $uri/ @fallback;
}

location @fallback {
  # If not a file, serve index.html (for client-side routing)
  root /app/frontend/build;
  rewrite ^(.*)$ /index.html break;
}

location /api/ {
  # Only things starting with /api go to backend
  proxy_pass http://api_backend;
}
```

Benefits:
- No Python process handles static files
- Nginx is faster at serving static content
- Can use CDN later (Cloudflare, AWS CloudFront)

## Real Configuration: PDF Service

```nginx
http {
  # Rate limiting
  limit_req_zone $binary_remote_addr zone=general:10m rate=100r/m;
  limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;
  limit_req_zone $binary_remote_addr zone=upload:10m rate=10r/m;
  
  # Upstream backends
  upstream api_backend {
    least_conn;
    server api1:8000;
    server api2:8000;
  }
  
  upstream celery_flower {
    server flower:5555;
  }
  
  # HTTP redirect to HTTPS
  server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
  }
  
  # HTTPS server
  server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # File upload size limit
    client_max_body_size 100M;
    
    # Static files (React)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
      root /app/frontend/build;
      expires 1y;
      add_header Cache-Control "public, immutable";
    }
    
    location / {
      root /app/frontend/build;
      try_files $uri $uri/ @api;
    }
    
    # API proxy
    location @api {
      limit_req zone=general burst=50 nodelay;
      
      proxy_pass http://api_backend;
      
      # Pass headers
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Request-ID $request_id;
      
      # Timeouts
      proxy_connect_timeout 60s;
      proxy_send_timeout 60s;
      proxy_read_timeout 60s;
    }
    
    # Auth endpoints: stricter rate limit
    location /api/auth/ {
      limit_req zone=auth burst=5 nodelay;
      proxy_pass http://api_backend;
      proxy_set_header Host $host;
    }
    
    # Upload endpoints: medium rate limit
    location /api/upload/ {
      limit_req zone=upload burst=10 nodelay;
      proxy_pass http://api_backend;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Monitoring (internal only)
    location /admin/flower/ {
      auth_basic "Restricted";
      auth_basic_user_file /etc/nginx/.htpasswd;
      
      proxy_pass http://celery_flower/;
    }
    
    # Health check
    location = /health {
      access_log off;
      proxy_pass http://api_backend/health;
    }
  }
}
```

## Common Proxy Headers

```nginx
proxy_set_header Host $host;
# Tell backend which domain was requested

proxy_set_header X-Real-IP $remote_addr;
# Client's true IP (not proxy's IP)

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
# List of IPs (client, proxy1, proxy2, ...)

proxy_set_header X-Forwarded-Proto $scheme;
# Was the request HTTP or HTTPS?

proxy_set_header X-Request-ID $request_id;
# For request tracing
```

Your FastAPI app should trust these headers:

```python
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware import Middleware

# Trust Nginx when it says the request came via HTTPS
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com"])

# Get client IP from X-Real-IP
@app.get("/api/documents")
async def get_client_ip(request: Request):
    client_ip = request.headers.get("X-Real-IP", request.client.host)
    return {"ip": client_ip}
```

## Hands-On Lab

### Lab 4.1: Load Balance Two API Instances

### docker-compose.yml
```yaml
version: '3'
services:
  api1:
    image: my-api:latest
    ports:
      - "8001:8000"
    environment:
      - PORT=8000

  api2:
    image: my-api:latest
    ports:
      - "8002:8000"
    environment:
      - PORT=8000

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api1
      - api2
```

### nginx.conf
```nginx
http {
  upstream api {
    server api1:8000;
    server api2:8000;
  }
  
  server {
    listen 80;
    
    location / {
      proxy_pass http://api;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

### Test
```bash
docker-compose up -d

# Make requests
curl http://localhost/api/documents
# Check which API instance handled it (logs or response)

# Add some delay to one API
# Observe load balancing in action
```

### Lab 4.2: Rate Limiting

Add to nginx.conf:
```nginx
http {
  limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;
  
  server {
    location / {
      limit_req zone=api burst=10 nodelay;
      proxy_pass http://api;
    }
  }
}
```

Test:
```bash
# Make 6 requests per second
for i in {1..20}; do
  curl http://localhost/ &
  sleep 0.1
done

# After 16 total requests (5r/s + 10 burst), you'll see:
# HTTP 429 Too Many Requests
```

## Cheat Sheet: Nginx

### Basic Proxy

```nginx
location /api/ {
  proxy_pass http://backend:8000;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
}
```

### Load Balance

```nginx
upstream backend {
  server server1:8000;
  server server2:8000;
}

location / {
  proxy_pass http://backend;
}
```

### Rate Limit

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location / {
  limit_req zone=api burst=20 nodelay;
  proxy_pass http://backend;
}
```

### Testing

```bash
nginx -t              # Syntax check
nginx -s reload       # Reload config (no downtime)
tail -f /var/log/nginx/error.log    # Debug
```

## Key Takeaways

- **Nginx = reverse proxy** — single entry point, routes to backends
- **SSL termination** = decrypt once at Nginx, forward unencrypted internally
- **Rate limiting** = prevent DDoS and abusive clients
- **Load balancing** = distribute requests across multiple backends
- **Static files** = serve from Nginx, skip the API
- **Proxy headers** = preserve client IP and protocol info

Module 5 teaches SSL/TLS certificates and security configuration.
