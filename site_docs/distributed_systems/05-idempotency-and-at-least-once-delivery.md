# Module 5: Idempotency and At-Least-Once Delivery

## The Problem: Retries Without Idempotency

API endpoint: transfer $100 from Account A to B.

Network splits:
```
API sends: "Transfer $100 from A to B"
PostgreSQL receives, processes, returns success
Network drops before API gets response
API times out after 30 seconds

Client thinks: "Request failed"
Client retries: "Transfer $100 from A to B" (again)

PostgreSQL: "You wrote success earlier? Oh, I'll do it again"
Result: $200 transferred instead of $100
```

**Issue**: Retry without idempotency = double-charge.

## Idempotency Definition

**Idempotent operation**: Calling it multiple times has the same result as calling once.

```
transfer(A, B, 100)         # Balance: A=900, B=1100
transfer(A, B, 100)         # Balance: A=900, B=1100 (unchanged)
transfer(A, B, 100)         # Balance: A=900, B=1100 (unchanged)
```

Non-idempotent:
```
increment(counter)          # counter=1
increment(counter)          # counter=2
increment(counter)          # counter=3
```

## Idempotency Keys

Solution: client sends a unique **idempotency key** with the request.

```
POST /api/transfer
{
  "idempotency_key": "uuid-1234",
  "from_account": "A",
  "to_account": "B",
  "amount": 100
}
```

Server:
1. Check if idempotency key exists in database
2. If yes, return cached result (same response as before)
3. If no, process request, cache result

```python
@app.post("/api/transfer")
def transfer(request: TransferRequest):
    idempotency_key = request.idempotency_key
    
    # Check cache
    cached = db.query(IdempotencyKey).filter_by(
        key=idempotency_key
    ).first()
    
    if cached:
        # Request already processed, return same response
        return json.loads(cached.response)
    
    # Process transfer
    from_acc = db.query(Account).filter_by(id=request.from_account).first()
    to_acc = db.query(Account).filter_by(id=request.to_account).first()
    
    from_acc.balance -= request.amount
    to_acc.balance += request.amount
    db.commit()
    
    # Cache result
    result = {"status": "transferred", "new_balance_a": from_acc.balance}
    idempotency_record = IdempotencyKey(
        key=idempotency_key,
        response=json.dumps(result),
        created_at=datetime.utcnow()
    )
    db.add(idempotency_record)
    db.commit()
    
    return result
```

**Now retries are safe**:

```
Request 1: idempotency_key=uuid-1234 → Transfer processed, cached
Request 2: idempotency_key=uuid-1234 → Return cached result (no double-transfer)
Request 3: idempotency_key=uuid-1234 → Return cached result
```

## At-Least-Once Delivery with Idempotency

Celery task can be executed multiple times (at-least-once).

```
Celery adds task: process_pdf_1 (task id = uuid-abc)
Worker picks up, executes
Worker crashes before acknowledging
Queue re-delivers: process_pdf_1

If code is not idempotent: PDF processed twice
If code is idempotent: PDF processed once
```

Make task idempotent:

```python
@app_celery.task
def process_pdf(pdf_id):
    pdf = db.query(PDF).get(pdf_id)
    
    # Idempotency: skip if already processed
    if pdf.status == "processed":
        return {"status": "already_processed"}
    
    # Do work
    result = extract_text(pdf)
    
    # Mark done
    pdf.status = "processed"
    pdf.content = result
    db.commit()
    
    return {"status": "processed", "pages": len(result)}
```

Even if Celery retries, idempotency prevents duplicate processing.

## UNIQUE Constraints for Idempotency

Database UNIQUE constraints enforce idempotency at the schema level.

### Example 1: Prevent Duplicate Subscriptions

```sql
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    gateway_reference VARCHAR UNIQUE NOT NULL,
    status VARCHAR,
    created_at TIMESTAMP DEFAULT NOW()
);
```

UNIQUE on `gateway_reference` prevents duplicate payments:

```python
# Webhook arrives twice with same reference
def process_payment_webhook(reference):
    try:
        payment = Payment(
            user_id=extract_user_id(),
            gateway_reference=reference,
            status="completed"
        )
        db.add(payment)
        db.commit()  # First time: success
    except IntegrityError:
        # Second webhook: UNIQUE constraint violation caught
        # Idempotent: subscription not created twice
        db.rollback()
        return {"status": "already_processed"}
```

### Example 2: Prevent Duplicate File Uploads

```sql
CREATE TABLE files (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    upload_idempotency_key VARCHAR UNIQUE,  -- Prevent duplicate uploads
    storage_path VARCHAR NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

User accidentally clicks "Upload" twice:

```python
@app.post("/api/files/upload")
def upload_file(file: UploadFile, idempotency_key: str):
    try:
        # Store file
        storage_id = str(uuid.uuid4())
        client.put_object("files", storage_id, file.file)
        
        # Record in database with idempotency key
        file_record = FileRecord(
            user_id=current_user.id,
            upload_idempotency_key=idempotency_key,
            storage_path=storage_id
        )
        db.add(file_record)
        db.commit()
        
        return {"id": file_record.id}
    except IntegrityError:
        # Duplicate upload attempt
        db.rollback()
        
        # Return ID of previously uploaded file
        existing = db.query(FileRecord).filter_by(
            upload_idempotency_key=idempotency_key
        ).first()
        return {"id": existing.id}
```

Second upload with same idempotency key returns first file's ID.

## Designing Idempotent Operations

### Rule 1: Read First, Then Write

```python
# Good: read current state, then modify
@app.post("/api/transfer")
def transfer(from_id, to_id, amount):
    from_account = db.query(Account).filter_by(id=from_id).first()
    current_balance = from_account.balance
    
    if current_balance < amount:
        raise HTTPException(status_code=400)
    
    new_balance = current_balance - amount
    from_account.balance = new_balance
    db.commit()
    
    # On retry: balance is already decreased, so same new_balance is set
```

### Rule 2: Use SET Operations, Not Increment

```python
# Bad: increment is not idempotent
db.execute(text("UPDATE account SET balance = balance - 100"))

# Good: set to specific value
db.execute(text("UPDATE account SET balance = :new_balance"), {"new_balance": 900})
```

### Rule 3: DELETE is Idempotent (Usually)

```python
# Delete by ID (safe, idempotent)
db.query(Document).filter_by(id=doc_id).delete()
db.commit()

# On retry: document already deleted, delete again does nothing
```

### Rule 4: Use Constraints

```sql
-- Idempotent schema
CREATE TABLE operations (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    operation_id VARCHAR UNIQUE,  -- Prevent duplicates
    operation_type VARCHAR,
    status VARCHAR,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Hands-On Lab

### Lab 5.1: Idempotency With Retries

```python
import uuid
import random

class TestTransfer:
    def __init__(self):
        self.db = {}  # Simulate database
    
    def transfer(self, idempotency_key, from_id, to_id, amount):
        """Non-idempotent transfer"""
        # Simulate network failure sometimes
        if random.random() < 0.3:
            raise Exception("Network error")
        
        from_bal = self.db.get(from_id, 0)
        to_bal = self.db.get(to_id, 0)
        
        if from_bal < amount:
            raise Exception("Insufficient funds")
        
        from_bal -= amount
        to_bal += amount
        
        self.db[from_id] = from_bal
        self.db[to_id] = to_bal
        
        return {"status": "ok", "balance": from_bal}
    
    def transfer_with_retries(self, idempotency_key, from_id, to_id, amount):
        """Retry with idempotency"""
        max_retries = 3
        for attempt in range(max_retries):
            try:
                return self.transfer(idempotency_key, from_id, to_id, amount)
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                print(f"Attempt {attempt+1} failed: {e}, retrying...")

# Test
test = TestTransfer()
test.db = {"A": 1000, "B": 0}

key = str(uuid.uuid4())
result = test.transfer_with_retries(key, "A", "B", 100)

print(f"Final balances: A={test.db['A']}, B={test.db['B']}")
# A=900 (not 880, 860, etc from multiple transfers)
```

### Lab 5.2: UNIQUE Constraint Enforcement

```python
# Create table with UNIQUE constraint
db.execute(text("""
CREATE TABLE IF NOT EXISTS idempotency_keys (
    id UUID PRIMARY KEY,
    key VARCHAR UNIQUE NOT NULL,
    result JSON NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
)
"""))

def idempotent_transfer(key, from_id, to_id, amount):
    """Idempotency via UNIQUE constraint"""
    try:
        # Perform transfer
        from_acc = 1000
        to_acc = 0
        
        from_acc -= amount
        to_acc += amount
        
        # Record result with idempotency key
        db.execute(text("""
            INSERT INTO idempotency_keys (id, key, result)
            VALUES (:id, :key, :result)
        """), {
            "id": str(uuid.uuid4()),
            "key": key,
            "result": json.dumps({
                "from_balance": from_acc,
                "to_balance": to_acc
            })
        })
        
        return {"status": "success"}
    except IntegrityError:
        # Duplicate key: return cached result
        existing = db.execute(text("""
            SELECT result FROM idempotency_keys WHERE key = :key
        """), {"key": key}).first()
        
        return json.loads(existing['result'])

# First call
key = str(uuid.uuid4())
result1 = idempotent_transfer(key, "A", "B", 100)
print(result1)  # {"status": "success"}

# Second call with same key
result2 = idempotent_transfer(key, "A", "B", 100)
print(result2)  # Same result, no double transfer
```

## Cheat Sheet: Idempotency

```python
# Client sends idempotency key
POST /api/transfer
{
  "idempotency_key": "uuid-1234",
  "from": "A",
  "to": "B",
  "amount": 100
}

# Server checks cache
cached = db.query(Idempotency).filter_by(key=key).first()
if cached:
    return cached.result

# Process and cache result
perform_transfer()
cache_result(key, result)

# Database UNIQUE constraint (alternative)
CREATE TABLE payments (
    gateway_reference VARCHAR UNIQUE
);

# On duplicate: IntegrityError caught, return cached result
```

## Key Takeaways

- **Idempotency** = calling multiple times = same result as calling once
- **Idempotency keys** prevent retries from duplicating work
- **At-least-once delivery** (Celery, webhooks) requires idempotent code
- **UNIQUE constraints** enforce idempotency at database layer
- **READ → MODIFY → WRITE** pattern is idempotent (not increment)
- **UNIQUE on business key** (gateway_reference, upload_key) prevents duplicates
- **Cache result** for idempotency key lookups

Module 6 teaches eventual consistency, the long-term sync pattern.
