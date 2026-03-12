# Module 2: Encryption Fundamentals

## The Analogy: The Sealed Safe

**Encryption** = lock data in a safe with a key
- Only person with the key can open it
- If lost, the data is gone forever (there's a trade-off)

**Hashing** = mix the data in a blender permanently
- Cannot be reversed (that's the point)
- When you need to verify (password), you blend the guess and compare

Two different tools for different problems.

## Symmetric vs Asymmetric

### Symmetric: One Key

Same key encrypts and decrypts.

```
Key: "my_secret_12345"

Encrypt: "password123" + key → "xykdJdj..."
Decrypt: "xykdJdj..." + key → "password123"
```

Algorithm: AES-256 (Advanced Encryption Standard)

Advantages:
- Fast
- Simple

Disadvantages:
- Key must be secret and shared somehow (how to send it safely?)

Use case: Data at rest (disk encryption, MinIO storage)

### Asymmetric: Public/Private Pair

Two mathematically linked keys: public (share freely) and private (keep secret).

```
Public key: (share with anyone)
  "encrypt_me_please@example.com"

Private key: (keep secret)
  "only_you_have_this"

Encrypt with public key → Only private key can decrypt
```

Advantages:
- Public key can be shared (no risk)
- Private key never leaves your server

Disadvantages:
- Much slower than symmetric

Use case: TLS handshake (secure key exchange before using fast symmetric crypto)

## Hashing: One-Way Encryption

Hash is irreversible. You cannot decrypt a hash.

```
Input: "password123"
Output: "$2y$12$x4...$7u..."  (bcrypt hash)
```

Try to hash again with same input:
```
Input: "password123"
Output: "$2y$12$x4...$7u..."  (same!)
```

Try different input:
```
Input: "password124"
Output: "$2y$12$y3...$8v..."  (completely different!)
```

Use case: Password storage. When user logs in, hash their guess and compare.

## Passwords: Hash with bcrypt

Never store plain passwords. Always hash.

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# When user registers
password = "MyPassword123!"
hashed_password = pwd_context.hash(password)
# Store hashed_password in database

# When user logs in
guess = "MyPassword123!"
if pwd_context.verify(guess, hashed_password):
    # Correct!
else:
    # Wrong password
```

### Why Not MD5 or SHA-1?

These are fast, non-salted hashes:

```
MD5("password123") = "482c811da5d5b4bc6d497ffa98491e38"
```

Problems:
- Attackers can pre-compute hashes of common passwords (rainbow tables)
- Finding collisions is easy
- Too fast (billions of guesses/second)

### Why Bcrypt?

```
bcrypt("password123", work_factor=12) = "$2y$12$KIXxPfxVW5x.tzJ..."
```

Advantages:
- **Salted**: Random salt added before hashing (no rainbow tables)
- **Slow by design**: Takes 0.5 seconds per hash (limits brute-force)
- **Work factor adjustable**: Increase as computers get faster

Trade-off: 0.5 second per password hash is slow, but that's the point. Attacker can only try ~2 passwords per second, even with a GPU.

```python
# Create hash with work_factor=12 (0.5s per hash)
hash_12 = pwd_context.hash("password", rounds=12)

# Create hash with work_factor=10 (0.1s per hash)
hash_10 = pwd_context.hash("password", rounds=10)

# hash_12 is slower but more secure
```

## Data at Rest: AES-256

Encrypt sensitive files on disk.

```python
from cryptography.fernet import Fernet

# Generate a key (do this once, store securely)
key = Fernet.generate_key()
# b'...'

cipher = Fernet(key)

# Encrypt data
plaintext = b"secret document content"
ciphertext = cipher.encrypt(plaintext)
# b'gAAAAABc...'

# Decrypt data
decrypted = cipher.decrypt(ciphertext)
# b"secret document content"
```

In production, use AWS KMS or HashiCorp Vault to store encryption keys.

MinIO can use server-side encryption:
```bash
# MinIO server starts with encryption enabled
export MINIO_KMS_KES_ENDPOINT=https://kes:7373
export MINIO_KMS_KES_KEY_FILE=/root/.kes/client.key
export MINIO_KMS_KES_CERT_FILE=/root/.kes/client.crt
export MINIO_KMS_KES_KEY_NAME=my-key

minio server /data
```

Files are encrypted at rest. Nobody, not even you, can access the raw data without the key.

## TLS: Asymmetric + Symmetric

TLS uses both:

```
1. Client + Server use asymmetric crypto (RSA key exchange)
   → Both agree on a symmetric key safely

2. All data uses symmetric crypto (AES-256)
   → Fast encryption/decryption

3. Keys are ephemeral (change per connection)
   → If one session is compromised, others are safe
```

See the networking section (Module 5) for details.

## Never Roll Your Own Crypto

Do NOT implement encryption yourself. Use battle-tested libraries:
- **bcrypt**: Password hashing
- **Python cryptography library**: AES, RSA, hashing
- **NaCl (libsodium)**: Modern crypto library
- **AWS KMS**: Key management (trusted)

Reasons:
- Timing attacks (implementation leaks via execution time)
- Side-channel attacks (power consumption, memory access patterns)
- Algorithm flaws (the crypto researchers are smarter than you)

## Hands-On Lab

### Lab 2.1: Hash a Password with bcrypt

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Test with different work factors
import time

for rounds in [4, 8, 12]:
    start = time.time()
    hash_val = pwd_context.hash("password123", rounds=rounds)
    elapsed = time.time() - start
    print(f"Rounds {rounds}: {elapsed:.2f}s")

# Output:
# Rounds 4: 0.02s
# Rounds 8: 0.15s
# Rounds 12: 0.50s

# Verify password
hash_12 = pwd_context.hash("password123", rounds=12)
if pwd_context.verify("password123", hash_12):
    print("Password matches!")

if not pwd_context.verify("wrong_password", hash_12):
    print("Wrong password correctly rejected")
```

### Lab 2.2: Encrypt/Decrypt Data

```python
from cryptography.fernet import Fernet

# Generate key (do this once, keep it safe)
key = Fernet.generate_key()
print(f"Key: {key}")  # Save this somewhere secure

cipher = Fernet(key)

# Encrypt
plaintext = b"My credit card: 1234 5678 9012 3456"
ciphertext = cipher.encrypt(plaintext)
print(f"Encrypted: {ciphertext}")

# Decrypt
decrypted = cipher.decrypt(ciphertext)
print(f"Decrypted: {decrypted}")

# If key is lost, data is unrecoverable
# This is intentional
```

## Cheat Sheet: Encryption and Hashing

### Password Hashing with bcrypt

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Hash password (on registration/password change)
hashed = pwd_context.hash("user_password")

# Verify password (on login)
if pwd_context.verify(user_input, hashed):
    # Correct!
```

### Symmetric Encryption (Data at Rest)

```python
from cryptography.fernet import Fernet

key = Fernet.generate_key()  # Generate once, store securely
cipher = Fernet(key)

ciphertext = cipher.encrypt(b"secret data")
plaintext = cipher.decrypt(ciphertext)
```

### Timing-Safe Comparison

```python
from hmac import compare_digest

# Bad (vulnerable to timing attacks)
if password == user_input:
    pass

# Good (constant time, immune to timing attacks)
if compare_digest(password, user_input):
    pass
```

## Key Takeaways

- **Symmetric encryption** = one key for both encrypt and decrypt (fast but key must be shared)
- **Asymmetric encryption** = public key encrypts, private key decrypts (slower but key sharing is safe)
- **Hashing** = one-way, irreversible, used for passwords
- **bcrypt** = salted, slow-by-design hashing (best for passwords)
- **AES-256** = symmetric encryption (use for data at rest)
- **TLS** = combines asymmetric (key exchange) and symmetric (data transfer)
- **Never roll your own** = use proven libraries
- **Never store plain passwords** = always hash immediately

Module 3 teaches common attacks and how to stop them.
