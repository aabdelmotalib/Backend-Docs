# Module 5: Payment Security and Webhooks

## Golden Rule: Never Touch Credit Cards

**Never**. Ever. Store or handle raw credit card numbers.

This triggers PCI-DSS (Payment Card Industry Data Security Standard) compliance:
- Requires annual audits ($50k+)
- Requires SOC 2 certification
- Massive liability

Instead: use a payment gateway (Paymob, Stripe, Square).

## Payment Flow: Your Platform

Your PDF platform uses Paymob.

```
1. User clicks "Subscribe"
2. Frontend redirects to Paymob payment page (hosted by Paymob, not you)
3. Paymob handles card collection securely
4. Paymob redirects back to your app with payment reference
5. Webhook: Paymob notifies you of payment status
6. Your app activates subscription
```

You never see credit card numbers.

## Webhook Verification: HMAC-SHA512

Webhook: POST request from Paymob to your app after payment.

**Problem**: Attacker can spoof webhooks.
```
POST /api/webhooks/payment
{
  "event": "payment.success",
  "amount_cents": 9999,  // $99.99
  "reference": "ref_123"
}
```

Attacker sends this without actually paying. You activate subscription for free.

**Solution**: HMAC signature verification.

Paymob signs each webhook:
```
X-HMAC-SHA512: hmac_sha512("secret_key" + body) = "abcd1234..."
```

You verify the signature matches. Tampering is detected.

### Implementation

```python
import hmac
import hashlib
import json

PAYMOB_SECRET = "your_webhook_secret"

@app.post("/api/webhooks/paymob")
async def handle_paymob_webhook(request: Request):
    body = await request.body()
    signature = request.headers.get("X-HMAC-SHA512")
    
    # Compute HMAC
    expected = hmac.new(
        PAYMOB_SECRET.encode(),
        body,
        hashlib.sha512
    ).hexdigest()
    
    # Timing-safe comparison
    if not hmac.compare_digest(signature, expected):
        raise HTTPException(status_code=403, detail="Invalid signature")
    
    # Signature valid, process webhook
    data = json.loads(body)
    process_payment(data)
    
    return {"status": "ok"}
```

Why `hmac.compare_digest`? It compares in constant time, preventing timing attacks.

**Attacker tries**:
```
X-HMAC-SHA512: abcd (wrong)
```

If code does `signature == expected`, timing reveals how many bytes matched:
- Wrong first byte: 10ms
- Wrong 10th byte: 100ms

Attacker learns bytes one by one. `compare_digest` always takes same time, regardless.

## The Double-Webhook Problem

Paymob might send the same webhook twice (network retry).

```
Webhook 1: payment_reference = "ref_123", activated subscription
Webhook 2 (retry): payment_reference = "ref_123", activated subscription AGAIN!
```

User charged twice.

**Solution**: Idempotency (Module 5.5 covers this fully).

```python
def process_payment(data):
    gateway_reference = data["reference"]
    
    # Check if already processed
    existing = db.query(Payment).filter_by(
        gateway_reference=gateway_reference
    ).first()
    if existing:
        return {"status": "already_processed"}
    
    # Create payment record
    payment = Payment(
        gateway_reference=gateway_reference,
        amount_cents=data["amount_cents"],
        user_id=extract_user_id(data)
    )
    db.add(payment)
    db.commit()
    
    # Activate subscription
    activate_subscription(payment.user_id)
    
    return {"status": "processed"}
```

The UNIQUE constraint on `gateway_reference` prevents duplicates:

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    gateway_reference VARCHAR UNIQUE NOT NULL,
    amount_cents INT NOT NULL,
    user_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

Even if webhook is processed twice, the second attempt hits the UNIQUE constraint. Triggers an error, but subscription is not activated twice. Idempotent.

## Never Activate from Frontend

Dangerous pattern:

```javascript
// ❌ NEVER DO THIS
fetch("/api/subscribe", {
  method: "POST",
  body: JSON.stringify({
    amount: 9999,
    currency: "USD"
  })
})
.then(r => r.json())
.then(data => {
  // User can modify this locally!
  // Add payment success immediately
  setUserSubscriptionActive(true);
})
```

User opens DevTools, calls this endpoint without payment. Subscription activated.

**Right way**: Wait for webhook.

```javascript
// ✅ CORRECT
// Frontend initiates payment redirect to Paymob
// After payment, Paymob redirects back
// Frontend polls backend: GET /api/me
// If subscription is active (set by webhook), show success

async function checkStatus() {
  const res = await fetch("/api/me");
  const { subscription_active } = await res.json();
  if (subscription_active) {
    showSuccess();
  }
}

// Poll every 2 seconds for 60 seconds
for (let i = 0; i < 30; i++) {
  await checkStatus();
  if (subscriptionActive) break;
  await sleep(2000);  // Wait 2 seconds
}
```

Subscription is activated only by webhook from Paymob. Frontend cannot fake it.

## Payment Webhook Handler: Complete

```python
import hmac
import hashlib
import json
from decimal import Decimal

PAYMOB_SECRET = "webhook_secret"
PAYMOB_WEBHOOK_KEY = "hmac"  # Header key

@app.post("/api/webhooks/paymob")
async def handle_paymob_webhook(request: Request):
    """Verify and process Paymob payment webhook"""
    
    body = await request.body()
    signature = request.headers.get(PAYMOB_WEBHOOK_KEY, "")
    
    # 1. Verify HMAC signature
    expected = hmac.new(
        PAYMOB_SECRET.encode(),
        body,
        hashlib.sha512
    ).hexdigest()
    
    if not hmac.compare_digest(signature, expected):
        logger.warning(f"Invalid HMAC: {signature}")
        raise HTTPException(status_code=403)
    
    # 2. Parse webhook
    data = json.loads(body)
    
    # 3. Check if already processed (idempotency)
    gateway_ref = data.get("gateway_reference")
    
    existing_payment = db.query(Payment).filter_by(
        gateway_reference=gateway_ref
    ).first()
    
    if existing_payment:
        # Already processed, return success
        logger.info(f"Webhook already processed: {gateway_ref}")
        return {"status": "ok", "already_processed": True}
    
    # 4. Verify payment success
    if data.get("success") != True:
        logger.warning(f"Payment failed: {gateway_ref}")
        return {"status": "ok"}  # Ack webhook but don't activate
    
    # 5. Extract user and amount
    user_id = data.get("user_id")  # You encode this in the request
    amount_cents = data.get("amount_cents")
    
    # 6. Create payment record (triggers UNIQUE constraint, prevents duplicates)
    try:
        payment = Payment(
            id=uuid.uuid4(),
            user_id=user_id,
            gateway_reference=gateway_ref,
            amount_cents=amount_cents,
            status="completed"
        )
        db.add(payment)
        db.commit()
    except IntegrityError:
        # Gateway reference already exists (duplicate webhook)
        db.rollback()
        logger.info(f"Duplicate webhook: {gateway_ref}")
        return {"status": "ok"}
    
    # 7. Activate subscription
    user = db.query(User).filter_by(id=user_id).first()
    user.is_subscribed = True
    user.subscription_expires_at = datetime.utcnow() + timedelta(days=365)
    db.commit()
    
    # 8. Send confirmation email
    send_subscription_confirmation(user)
    
    logger.info(f"Payment processed: {gateway_ref}")
    return {"status": "ok"}
```

## Hands-On Lab

### Lab 5.1: HMAC Verification

```python
import hmac
import hashlib

secret = "my_secret"
message = "payment_success"

# Sign
signature = hmac.new(
    secret.encode(),
    message.encode(),
    hashlib.sha512
).hexdigest()

print(f"Signature: {signature}")

# Verify (correct)
expected = hmac.new(
    secret.encode(),
    message.encode(),
    hashlib.sha512
).hexdigest()

if hmac.compare_digest(signature, expected):
    print("✓ Signature valid")

# Verify (attacker tampered)
tampered_sig = signature[:-1] + "x"
if not hmac.compare_digest(tampered_sig, expected):
    print("✓ Tampering detected")
```

### Lab 5.2: Idempotent Webhook Handler

```python
# Simulate webhook arriving twice

def process_webhook(payment_ref):
    existing = db.query(Payment).filter_by(
        gateway_reference=payment_ref
    ).first()
    
    if existing:
        return "already_processed"
    
    payment = Payment(gateway_reference=payment_ref)
    db.add(payment)
    db.commit()
    
    return "processed"

# First call
print(process_webhook("ref_123"))  # "processed"

# Second call (webhook retry)
print(process_webhook("ref_123"))  # "already_processed"

# Subscription was activated only once
```

## Cheat Sheet: Payment Security

```python
# HMAC verification
import hmac, hashlib
signature = request.headers.get("X-HMAC")
expected = hmac.new(SECRET.encode(), body, hashlib.sha512).hexdigest()
if not hmac.compare_digest(signature, expected):
    raise HTTPException(status_code=403)

# Idempotency via UNIQUE constraint
existing = db.query(Payment).filter_by(
    gateway_reference=gateway_ref
).first()
if existing:
    return {"status": "ok"}

# Never activate from frontend
# Only activate via webhook verified above

# Database schema
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    gateway_reference VARCHAR UNIQUE NOT NULL,
    amount_cents INT NOT NULL,
    user_id UUID NOT NULL,
    status VARCHAR,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Golden Rules

1. **Never touch raw credit cards** — use payment gateway
2. **Always verify HMAC** — detect tampering
3. **Use UNIQUE constraint** — prevent duplicate activation
4. **Compare with constant time** — use `hmac.compare_digest`
5. **Never activate from frontend** — only from verified webhook
6. **Log everything** — recreate payments from logs

## Key Takeaways

- **Payment gateways** handle cards securely (Paymob, Stripe)
- **HMAC-SHA512** verifies webhook authenticity
- **Timing-safe comparison** prevents timing attacks
- **UNIQUE gateway_reference** ensures idempotency
- **Webhooks only** activate subscriptions, not frontend
- **PCI-DSS** compliance automatic when using payment gateway

This completes the Security section. Next: Distributed Systems —how services coordinate without being in one place.
