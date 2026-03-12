# Networking: Internet Fundamentals for Backend Engineers

## Why Networking Matters

You cannot build a backend without networking. Your API lives on the internet. Understanding networking means:
- **Diagnosing production issues**: "Why is the API slow?" Often it's not the code, it's the network.
- **Securing infrastructure**: Firewalls, TLS, DNS — all network concepts.
- **Designing with latency in mind**: If you cannot measure latency, you cannot optimize.
- **Debugging with real tools**: ping, traceroute, curl, dig — understanding what they show.

A backend engineer who doesn't understand networking is like a pilot who doesn't understand aerodynamics.

## What You'll Learn

### 1. How the Internet Works
IP addresses, ports, TCP vs UDP, DNS resolution, subnets, request journey
- **Lab**: Trace a request from your PC to a remote server using traceroute

### 2. HTTP and HTTPS
Request/response anatomy, status codes, headers, REST design, TLS
- **Lab**: Inspect full HTTP exchange with curl -v flag

### 3. DNS Explained
Domain names, A/CNAME/MX records, TTL, Let's Encrypt verification
- **Lab**: Point your domain to your server and verify propagation

### 4. Nginx Reverse Proxy
Rate limiting, SSL termination, static files, load balancing, real config
- **Lab**: Load balance across two API instances

### 5. SSL/TLS and Certificates
What TLS encrypts, certificates, Let's Encrypt + Certbot, HSTS, modern ciphers
- **Lab**: Obtain and install a free certificate, verify with openssl

### 6. Firewalls and Ports
ufw configuration, hardening SSH, Fail2ban, the principle of least privilege
- **Lab**: Open only 22/80/443, verify internal ports are unreachable

## The PDF Service Platform Architecture

```
                        ┌─────────────────────┐
                        │  Browser / Client   │
                        └──────────────┬──────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │  Nginx Reverse Proxy        │
                        │  (SSL termination)          │
                        │  (Rate limiting)            │
                        │  (Static files)             │
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │  FastAPI Backend            │
                        │  (HTTP/HTTPS only)          │
                        └──────────────┬──────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
            ┌───────▼──────┐  ┌────────▼────────┐  ┌─────▼────────┐
            │  PostgreSQL  │  │  Redis Cache    │  │  MinIO S3    │
            │  (5432)      │  │  (6379)         │  │  (9000)      │
            │  Private     │  │  Private        │  │  Private     │
            └──────────────┘  └─────────────────┘  └──────────────┘
```

**Key insight**: Only 80/443 (Nginx) are exposed to the internet. Everything else (5432, 6379, 9000) is blocked by the firewall.

## Prerequisites

- Basic understanding of how the web works
- SSH access to a server
- A domain name (for DNS and certificate modules)
- Familiarity with command line

## How to Use This Section

- **Module 1** is fundamental — read it before anything else
- **Modules 2-3** are prerequisite for 4-6
- **Module 4** can be read after module 1, alongside 2-3
- **Modules 5-6** assume you understand modules 1-2

Each module includes:
- Non-technical analogy first
- Technical explanation with real code/config
- Real examples from the PDF platform
- Hands-on lab you can run immediately
- Cheat sheet for reference

Let's start with the fundamental question: how does the internet actually work?
