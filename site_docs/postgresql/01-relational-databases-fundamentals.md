# Module 1: Relational Database Fundamentals

## The Analogy: A Library Card Catalog

Imagine a library card catalog (the old system with drawers):

**Before**: All information about a book on one card.
```
Title: Advanced Python
Author: Guido van Rossum
ISBN: 12345
Subject: Programming
Location: Shelf A5
Checkout Date: 2024-01-15
Due Date: 2024-02-15
Borrower: Alice Smith
Borrower Address: 123 Main St
Borrower Phone: 555-1234
```

This is **denormalized**. If Alice moves, you update 50 cards.

**After**: Separate drawers (tables) for Books, Borrowers, Checkouts.

```
BOOKS:
├─ id: 1
├─ title: Advanced Python
├─ author_id: 5

AUTHORS:
├─ id: 5
├─ name: Guido van Rossum

BORROWERS:
├─ id: 10
├─ name: Alice Smith
├─ address: 123 Main St
├─ phone: 555-1234

CHECKOUTS:
├─ id: 100
├─ book_id: 1
├─ borrower_id: 10
├─ due_date: 2024-02-15
```

Now if Alice moves, you update ONE address. Related records automatically reflect the change. This is **normalization** — the foundation of relational databases.

## Tables and Columns

A **table** is a collection of related data, organized in rows and columns.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);
```

This creates a `users` table with:
- `id` — unique identifier (primary key)
- `email` — up to 255 characters, required
- `name` — up to 100 characters, optional (can be NULL)
- `created_at` — timestamp, defaults to current time

## Primary Keys and Foreign Keys

### Primary Key

A **primary key** uniquely identifies each row.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

Every user has a unique `id`. You cannot have two users with `id=5`.

### Foreign Key

A **foreign key** links to another table.

```sql
CREATE TABLE subscriptions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    plan VARCHAR(50),
    active BOOLEAN DEFAULT TRUE
);
```

The `user_id` column **references** the `users` table. You cannot create a subscription with `user_id=999` if user 999 doesn't exist.

### Relationship Example

```
users:
├─ id=1, email=alice@example.com
├─ id=2, email=bob@example.com

subscriptions:
├─ id=101, user_id=1, plan="pro"
├─ id=102, user_id=1, plan="pro"
├─ id=103, user_id=2, plan="free"
```

User 1 (Alice) has 2 subscriptions. User 2 (Bob) has 1.

Query: "Show me all subscriptions for user 1":

```sql
SELECT * FROM subscriptions WHERE user_id = 1;
```

Result:
```
id  user_id  plan
101  1       pro
102  1       pro
```

## Constraints: Enforcing Data Quality

| Constraint | Purpose | Example |
|-----------|---------|---------|
| PRIMARY KEY | No duplicates | `id SERIAL PRIMARY KEY` |
| UNIQUE | No duplicates (non-id) | `email VARCHAR(255) UNIQUE` |
| NOT NULL | Must have a value | `email VARCHAR(255) NOT NULL` |
| FOREIGN KEY | Link to another table | `user_id REFERENCES users(id)` |
| CHECK | Value must meet condition | `age INTEGER CHECK (age >= 0)` |
| DEFAULT | Use this if not specified | `created_at TIMESTAMP DEFAULT NOW()` |

### Real Example: Payments Table

```sql
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    subscription_id INTEGER NOT NULL REFERENCES subscriptions(id),
    amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (amount > 0),
    CHECK (status IN ('pending', 'completed', 'failed'))
);
```

Constraints:
- `amount > 0` — cannot charge negative amounts
- `status` must be one of three values
- `subscription_id` must exist in `subscriptions` table
- `created_at` defaults to now if not provided

## Data Types

Common PostgreSQL types:

| Type | Purpose | Example |
|------|---------|---------|
| SERIAL | Auto-incrementing integer | `id SERIAL PRIMARY KEY` |
| INTEGER | Whole numbers | `age INTEGER` |
| DECIMAL(10,2) | Money | `amount DECIMAL(10,2)` |
| VARCHAR(n) | Text up to n chars | `email VARCHAR(255)` |
| TEXT | Long text | `description TEXT` |
| BOOLEAN | True/False | `active BOOLEAN` |
| TIMESTAMP | Date and time | `created_at TIMESTAMP` |
| DATE | Date only | `birthdate DATE` |
| JSON | Nested data | `metadata JSON` |
| UUID | 36-char identifier | `id UUID PRIMARY KEY` |

## SQL Basics: CRUD

### CREATE (Insert)

```sql
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice');
INSERT INTO users (email, name) VALUES ('bob@example.com', 'Bob');
```

### READ (Select)

```sql
-- All columns, all rows
SELECT * FROM users;

-- Specific columns
SELECT id, email FROM users;

-- With condition
SELECT * FROM users WHERE id = 1;

-- With multiple conditions
SELECT * FROM users WHERE id > 5 AND email LIKE '%@example.com%';

-- Sorted
SELECT * FROM users ORDER BY created_at DESC;

-- Limited
SELECT * FROM users LIMIT 10;
```

### UPDATE (Modify)

```sql
UPDATE users SET name = 'Alice Smith' WHERE id = 1;

UPDATE subscriptions SET active = FALSE WHERE user_id = 2;
```

### DELETE (Remove)

```sql
DELETE FROM users WHERE id = 1;

-- Dangerous! Deletes all inactive subscriptions
DELETE FROM subscriptions WHERE active = FALSE;
```

!!! warning
    Always be careful with DELETE. Add a WHERE clause. Test with SELECT first.

## Joins: Querying Across Tables

### INNER JOIN

"Give me users AND their subscriptions"

```sql
SELECT u.email, s.plan
FROM users u
INNER JOIN subscriptions s ON u.id = s.user_id;
```

Result (only rows where user has subscription):
```
email                plan
alice@example.com    pro
alice@example.com    pro
bob@example.com      free
```

### LEFT JOIN

"Give me all users, and their subscriptions if they have any"

```sql
SELECT u.email, s.plan
FROM users u
LEFT JOIN subscriptions s ON u.id = s.user_id;
```

Result (includes users without subscriptions):
```
email                plan
alice@example.com    pro
alice@example.com    pro
bob@example.com      free
carol@example.com    (null)
```

## Aggregations: GROUP BY and COUNT

```sql
SELECT user_id, COUNT(*) as subscription_count
FROM subscriptions
GROUP BY user_id;
```

Result:
```
user_id  subscription_count
1        2
2        1
```

Common aggregates:
- `COUNT(*)` — number of rows
- `SUM(amount)` — total of column
- `AVG(amount)` — average
- `MAX(amount)` — highest value
- `MIN(amount)` — lowest value

```sql
SELECT 
    EXTRACT(YEAR FROM created_at) as year,
    SUM(amount) as total_revenue,
    COUNT(*) as payment_count
FROM payments
GROUP BY EXTRACT(YEAR FROM created_at);
```

## Hands-On Lab

### Lab 1.1: Create Tables

Create a PostgreSQL container:

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  postgres:15-alpine
```

Connect:

```bash
docker exec -it postgres psql -U postgres
```

Create the users table:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    plan VARCHAR(50) NOT NULL,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Lab 1.2: Insert and Query

Insert users:

```sql
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice');
INSERT INTO users (email, name) VALUES ('bob@example.com', 'Bob');

SELECT * FROM users;
```

Insert subscriptions:

```sql
INSERT INTO subscriptions (user_id, plan) VALUES (1, 'pro');
INSERT INTO subscriptions (user_id, plan) VALUES (1, 'pro');
INSERT INTO subscriptions (user_id, plan) VALUES (2, 'free');

SELECT * FROM subscriptions;
```

### Lab 1.3: Join Tables

Query across tables:

```sql
SELECT u.email, s.plan
FROM users u
INNER JOIN subscriptions s ON u.id = s.user_id;
```

Count subscriptions per user:

```sql
SELECT u.email, COUNT(s.id) as subscription_count
FROM users u
LEFT JOIN subscriptions s ON u.id = s.user_id
GROUP BY u.id, u.email;
```

## Cheat Sheet: SQL Basics

### Create Table

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Insert

```sql
INSERT INTO users (email) VALUES ('alice@example.com');
```

### Select

```sql
SELECT * FROM users;
SELECT * FROM users WHERE id = 1;
SELECT email FROM users ORDER BY created_at DESC;
```

### Update

```sql
UPDATE users SET name = 'Alice' WHERE id = 1;
```

### Delete

```sql
DELETE FROM users WHERE id = 1;
```

### Join

```sql
SELECT u.email, s.plan
FROM users u
INNER JOIN subscriptions s ON u.id = s.user_id;
```

### Group By

```sql
SELECT user_id, COUNT(*) as count
FROM subscriptions
GROUP BY user_id;
```

## Key Takeaways

- **Relational databases organize data into tables** — like spreadsheets with relationships
- **Primary keys uniquely identify rows** — each `id` is unique
- **Foreign keys link tables together** — enforce relationships
- **Constraints ensure data quality** — NOT NULL, CHECK, UNIQUE
- **SQL is the language** — SELECT, INSERT, UPDATE, DELETE
- **Joins query across tables** — INNER, LEFT, etc.
- **Normalization reduces duplication** — change once, affects all

Module 2 teaches installing PostgreSQL and connecting to it.
