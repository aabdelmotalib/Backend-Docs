# Module 4: Alembic Migrations

## The Analogy: Version Control for Database

Git version-controls your code:

```
commit 1: def hello(): print("Hello")
commit 2: def hello(): print("Hello World")
commit 3: Revert to commit 1
```

Alembic version-controls your database schema:

```
migration 1: CREATE TABLE users;
migration 2: ALTER TABLE users ADD COLUMN phone;
migration 3: Rollback to migration 1
```

Without migrations, if you change the code but not the database, your app breaks. With Alembic, your schema stays in sync.

## What Alembic Does

Alembic:

1. **Tracks schema changes** — Each change is a migration
2. **Applies changes** — Move forward (upgrade) or backward (downgrade)
3. **Documents changes** — Every migration is timestamped
4. **Prevents conflicts** — Know which migrations are applied

## Setup

### Install

```bash
pip install alembic
```

### Initialize Project

```bash
alembic init migrations
```

This creates:

```
migrations/
├── env.py          # Configuration
├── script.py.mako  # Migration template
├── versions/       # Folder for migrations
│   ├── 001_initial.py
│   ├── 002_add_phone.py
│   └── ...
└── alembic.ini     # Settings
```

### Configure alembic.ini

Set your database URL:

```ini
sqlalchemy.url = postgresql+asyncpg://user:pass@localhost/mydb
```

Or use environment variable:

```ini
sqlalchemy.url = driver://user:password@localhost/dbname
```

## Creating Migrations

### Autogenerate (Recommended)

Alembic compares your models to the database and generates migration:

```bash
alembic revision --autogenerate -m "Add phone to users"
```

Creates `versions/001_add_phone_to_users.py`:

```python
def upgrade() -> None:
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

def downgrade() -> None:
    op.drop_column('users', 'phone')
```

Review the migration before running it to ensure it's correct.

### Manual (For Complex Changes)

```bash
alembic revision -m "Custom logic"
```

Creates empty migration. You fill in the logic:

```python
def upgrade() -> None:
    # Your SQL here
    op.execute("UPDATE users SET phone = '555-0000' WHERE phone IS NULL")

def downgrade() -> None:
    op.execute("UPDATE users SET phone = NULL")
```

## Applying Migrations

### To Latest

```bash
alembic upgrade head
```

Applies all pending migrations.

### To Specific Migration

```bash
alembic upgrade 001_initial
```

Applies migrations up to `001_initial`.

### Downgrade (Rollback)

```bash
alembic downgrade -1  # Back one migration
alembic downgrade -5  # Back five migrations
alembic downgrade 001 # Back to migration 001
```

## Migration History

### View History

```bash
alembic history
```

Output:

```
<base> -> 001_initial, 2024-01-15 10:00:00
001_initial -> 002_add_phone, 2024-01-15 11:00:00
```

### Current Version

```bash
alembic current
```

Output:

```
INFO  [alembic.migration] Context impl PostgresqlImpl
2024_01_15_110000_add_phone (head)
```

## Real Workflow

### Step 1: Change Model

```python
# models.py
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String(255))
    phone = Column(String(20))  # NEW
    address = Column(String(500))  # NEW
```

### Step 2: Generate Migration

```bash
alembic revision --autogenerate -m "Add phone and address to users"
```

### Step 3: Review Migration

```bash
cat migrations/versions/001_add_phone_and_address.py
```

Review the changes:

```python
def upgrade() -> None:
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))
    op.add_column('users', sa.Column('address', sa.String(500), nullable=True))

def downgrade() -> None:
    op.drop_column('users', 'address')
    op.drop_column('users', 'phone')
```

### Step 4: Apply Migration

```bash
alembic upgrade head
```

### Step 5: Verify

```bash
docker exec postgres psql -U postgres -d mydb -c "\d users"
```

Should show the new columns.

## Data Migrations

Sometimes you need to transform data during migration:

```python
def upgrade() -> None:
    op.add_column('users', sa.Column('created_year', sa.Integer))
    
    # Set created_year from existing created_at
    op.execute("""
        UPDATE users 
        SET created_year = EXTRACT(YEAR FROM created_at)
    """)

def downgrade() -> None:
    op.drop_column('users', 'created_year')
```

## Database Constraints

Alembic can add constraints:

```python
def upgrade() -> None:
    # Add unique constraint
    op.create_unique_constraint('uq_users_email', 'users', ['email'])
    
    # Add check constraint
    op.create_check_constraint('ck_users_age', 'users', 'age >= 0')

def downgrade() -> None:
    op.drop_constraint('uq_users_email', 'users')
    op.drop_constraint('ck_users_age', 'users')
```

## In Production

### Workflow

1. **Develop locally** → Create models → Generate migrations → Test
2. **Commit** → Push code and migrations to git
3. **Review** → Check migrations in code review
4. **Deploy** → Run `alembic upgrade head` before starting new code version

### Safety

```bash
# Check what will run
alembic upgrade head --sql

# Do nothing yet; just show SQL
# Review carefully

# Then actually run
alembic upgrade head
```

### Backup Before Major Migrations

```bash
# Backup database
pg_dump production_db > backup_pre_migration.sql

# Run migration
alembic upgrade head

# If disaster: restore
psql production_db < backup_pre_migration.sql
```

## Hands-On Lab

### Lab 4.1: Initialize Alembic

```bash
mkdir -p alembic_project
cd alembic_project

python -m venv venv
source venv/bin/activate

pip install sqlalchemy alembic aiosqlite

alembic init migrations
```

### Lab 4.2: Create Models

Create `models.py`:

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True)
    name = Column(String(100))
```

### Lab 4.3: Generate First Migration

```bash
alembic revision --autogenerate -m "Initial schema"
```

Check `migrations/versions/` for generated migration.

### Lab 4.4: Apply Migration

```bash
# Check SQL that will run
alembic upgrade head --sql

# Apply
alembic upgrade head

# Verify
alembic current
alembic history
```

### Lab 4.5: Data Migration

Update `models.py`:

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True)
    name = Column(String(100))
    age = Column(Integer)  # NEW
```

Generate and check:

```bash
alembic revision --autogenerate -m "Add age to users"
```

## Cheat Sheet: Alembic Migrations

### Initialize

```bash
alembic init migrations
```

### Create Migration

```bash
# Auto-generate from model changes
alembic revision --autogenerate -m "description"

# Manual creation
alembic revision -m "description"
```

### Apply Migrations

```bash
alembic upgrade head         # All pending
alembic upgrade -1           # Back one
alembic downgrade -1         # Revert one
```

### Check Status

```bash
alembic current              # Current version
alembic history              # All migrations
alembic upgrade head --sql   # Preview SQL (no changes)
```

### In Migration File

```python
def upgrade() -> None:
    op.add_column('users', sa.Column('phone', sa.String(20)))
    op.create_index('ix_users_email', 'users', ['email'])

def downgrade() -> None:
    op.drop_column('users', 'phone')
    op.drop_index('ix_users_email')
```

## Key Takeaways

- **Alembic = version control for database schema** — like git but for tables
- **Models define schema** — Alembic tracks changes to models
- **Migrations are reversible** — upgrade and downgrade
- **Autogenerate** — let Alembic write migrations by comparing model to database
- **Review before applying** — use `--sql` flag to preview
- **Data migrations** — transform data when schema changes
- **Production safety** — backup before major migrations

Module 5 teaches Indexes — how to make queries fast.
