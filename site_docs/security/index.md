# Security: Protecting Your Users' Data

## Overview

Security is not optional. It's non-negotiable.

You handle:
- User passwords (must be hashed, never stored plain)
- Payment information (never store raw credit cards)
- File uploads (could contain malware)
- API tokens (could be stolen if intercepted)
- User data (subject to privacy laws like GDPR)

A breach costs money, trust, and time. Prevention is easier than recovery.

## The 5 Layers

### 1. Network Security
Only expose necessary ports. Use TLS. (Covered in networking section)

### 2. Authentication & Authorization
Who are you? What are you allowed to do?

### 3. Encryption
Data at rest (MinIO, database) and in transit (TLS).

### 4. Application Security
Stop SQL injection, XSS, CSRF, and other attacks.

### 5. Data Security
Validate uploads. Scan for malware. Protect payment processing.

## What You'll Learn

### Module 1: Authentication vs Authorization
Proving who you are, and what you're permitted to do.
- JWT tokens, stateless auth, OAuth2 overview

### Module 2: Encryption Fundamentals
Symmetric (same key) vs asymmetric (public/private). Hashing vs encryption.
- bcrypt for passwords, AES for data at rest

### Module 3: Common Attack Vectors
SQL injection, XSS, CSRF, path traversal, SSRF.
- Real examples, how to prevent each

### Module 4: Securing File Uploads
The most dangerous attack surface.
- Magic byte validation, size limits, virus scanning, UUID paths

### Module 5: Payment Security
PCI-DSS compliance without holding raw card data.
- Payment gateways, webhook signature verification, idempotency

## The PDF Service Security Model

```
┌─────────────────────────────────────────────────────┐
│ User Access Layer                                   │
│ - Authentication (JWT token)                       │
│ - Authorization (subscription level)               │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│ API Layer (FastAPI)                                │
│ - Input validation (Pydantic)                      │
│ - Rate limiting (Nginx)                            │
│ - CSRF protection (SameSite cookies)               │
└─────────────────┬───────────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼───┐   ┌────▼───┐   ┌───▼────┐
│File   │   │Database│   │Object  │
│Upload │   │(SQL    │   │Storage │
│Scan   │   │Inject) │   │(Path   │
│       │   │        │   │Traversal
└───────┘   └────────┘   └────────┘
```

## Prerequisites

- Understanding of HTTP/HTTPS (from Networking section)
- Familiarity with Python and FastAPI
- Basic SQL knowledge

## How to Use This Section

**Essential reading order**:
1. **Module 1**: Authentication vs Authorization (foundational)
2. **Module 2**: Encryption (understand before implementing)
3. **Module 3**: Attack Vectors (know what you're defending against)
4. **Module 4**: File Uploads (practical security)
5. **Module 5**: Payment Security (if your platform handles payments)

Each module:
- Explains the threat clearly
- Shows vulnerable code
- Shows secure code
- Provides hands-on labs
- Includes implementation checklist

## Golden Rules

1. **Never trust user input** — validate and sanitize everything
2. **Never store passwords in plain text** — use bcrypt
3. **Never store raw credit cards** — use payment gateways
4. **Never run uploaded files** — validate type and scan
5. **Always use HTTPS** — no exceptions
6. **Always hash passwords** — even "if no one will notice"
7. **Always verify webhooks** — HMAC signatures are non-negotiable

Let's start with the foundation: authentication and authorization.
