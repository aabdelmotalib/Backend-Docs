# Module 5: Bash Scripting for DevOps

## The Analogy: Recipes for Cooking

A bash script is a recipe:

- **Ingredients** are variables (`username="alice"`)
- **Instructions** are commands (`echo`, `grep`, `curl`)
- **Decisions** are conditionals (`if`, `case`)
- **Repetition** are loops (`for`, `while`)

You write a script once, run it 100 times. Automation saves time and prevents human error.

## Variables in Bash

### Setting Variables

```bash
name="Alice"
age=30
email="alice@example.com"

# No spaces around =
# Right: name="Alice"
# Wrong: name = "Alice"
```

### Using Variables

```bash
echo "Hello, $name"          # Hello, Alice
echo "I am $age years old"   # I am 30 years old
```

### Command Substitution

```bash
current_date=$(date +%Y-%m-%d)
echo "Today is $current_date"

# Or use backticks (older style)
current_date=`date +%Y-%m-%d`
```

## Conditionals: if/then/else

### Simple if

```bash
if [ $age -gt 18 ]; then
    echo "You are an adult"
fi
```

### if/else

```bash
if [ $age -gt 18 ]; then
    echo "You are an adult"
else
    echo "You are a minor"
fi
```

### if/elif/else

```bash
if [ $age -lt 13 ]; then
    echo "Child"
elif [ $age -lt 18 ]; then
    echo "Teenager"
else
    echo "Adult"
fi
```

### Comparison Operators

```bash
[ $a -eq $b ]     # Equal
[ $a -ne $b ]     # Not equal
[ $a -lt $b ]     # Less than
[ $a -le $b ]     # Less than or equal
[ $a -gt $b ]     # Greater than
[ $a -ge $b ]     # Greater than or equal

[ -f "$file" ]    # File exists
[ -d "$dir" ]     # Directory exists
[ -z "$var" ]     # String is empty
[ -n "$var" ]     # String is not empty
[ "$var" = "value" ]  # String equals
```

### Combining Conditions

```bash
if [ $age -gt 18 ] && [ "$status" = "active" ]; then
    echo "Eligible"
fi

if [ $error -eq 0 ] || [ $error -eq 1 ]; then
    echo "OK"
fi

if [ ! -f "$file" ]; then
    echo "File does not exist"
fi
```

## Loops: for, while

### for Loop

```bash
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

# Or with range
for i in {1..5}; do
    echo "Number: $i"
done

# Or with command output
for file in *.log; do
    echo "Processing $file"
done
```

### while Loop

```bash
counter=1
while [ $counter -le 5 ]; do
    echo "Count: $counter"
    counter=$((counter + 1))
done
```

## Functions

```bash
greet() {
    echo "Hello, $1"
}

greet "Alice"       # Output: Hello, Alice

# Functions with return values
add() {
    local result=$(($1 + $2))
    echo $result
}

sum=$(add 5 3)
echo "Sum: $sum"    # Output: Sum: 8
```

## Exit Codes and Error Handling

Every command returns an exit code:

- `0` = success
- Non-zero = failure

```bash
curl https://api.example.com
echo $?              # Print exit code (0 if successful)
```

### set -e: Exit on Error

```bash
#!/bin/bash
set -e  # Exit immediately if any command fails

curl https://api.example.com/data  # Stops here if this fails
echo "Done"                          # Won't run if curl fails
```

### Checking Exit Codes

```bash
if curl https://api.example.com > /dev/null 2>&1; then
    echo "API is up"
else
    echo "API is down"
fi
```

## Reading .env Files

Many apps use `.env` files for configuration:

```
DB_PASSWORD=secretpass
DB_HOST=localhost
DB_PORT=5432
```

Read them in bash:

```bash
# Method 1: Source the file
source .env
echo $DB_PASSWORD

# Method 2: Read line by line
while IFS='=' read -r key value; do
    export "$key=$value"
done < .env

# Method 3: Using grep and export
export $(grep -v '^#' .env | xargs)
```

## Real-World Example: Deploy Script

A realistic production deployment script:

```bash
#!/bin/bash
set -e

echo "[1/5] Loading configuration..."
if [ ! -f ".env" ]; then
    echo "Error: .env file not found"
    exit 1
fi
source .env

echo "[2/5] Pulling latest code..."
git pull origin main || {
    echo "Failed to pull code"
    exit 1
}

echo "[3/5] Building Docker images..."
docker compose build api worker || {
    echo "Failed to build images"
    exit 1
}

echo "[4/5] Starting containers..."
docker compose up -d --no-deps api worker || {
    echo "Failed to start containers"
    exit 1
}

echo "[5/5] Checking API health..."
max_attempts=30
attempt=0

while [ $attempt -lt $max_attempts ]; do
    if curl -f http://localhost:8000/health > /dev/null 2>&1; then
        echo "✓ API is healthy"
        exit 0
    fi
    
    echo "  Waiting for API... ($((attempt + 1))/$max_attempts)"
    sleep 1
    attempt=$((attempt + 1))
done

echo "✗ API failed to become healthy"
docker compose logs api
exit 1
```

## Real-World Example: Health Check Script

Monitor container health:

```bash
#!/bin/bash

SERVICES=("api" "postgres" "redis" "worker")
FAILED=0

echo "Checking service health..."

for service in "${SERVICES[@]}"; do
    # Check if container is running
    if ! docker compose ps $service | grep -q "Up"; then
        echo "✗ $service is not running"
        FAILED=$((FAILED + 1))
    else
        echo "✓ $service is running"
        
        # Check if healthy (if healthcheck exists)
        if docker compose ps $service | grep -q "(healthy)"; then
            echo "  └─ healthy"
        elif docker compose ps $service | grep -q "(unhealthy)"; then
            echo "  └─ UNHEALTHY!"
            FAILED=$((FAILED + 1))
        fi
    fi
done

if [ $FAILED -gt 0 ]; then
    echo ""
    echo "⚠️  $FAILED service(s) have issues"
    exit 1
else
    echo ""
    echo "✓ All services healthy"
    exit 0
fi
```

## Cron Jobs: Scheduled Automation

**cron** runs scripts on a schedule. Edit your crontab:

```bash
crontab -e
```

Add a line:

```
# Run backup.sh every day at 2 AM
0 2 * * * /home/alice/backup.sh

# Run health check every 5 minutes
*/5 * * * * /home/alice/check-health.sh

# Run cleanup every Sunday at 3 AM
0 3 * * 0 /home/alice/cleanup.sh
```

### Cron Syntax

```
┌─────────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌─────────── day of month (1-31)
│ │ │ ┌───────── month (1-12)
│ │ │ │ ┌─────── day of week (0-6, 0=Sunday)
│ │ │ │ │
│ │ │ │ │
* * * * * command
```

Examples:

```
0 2 * * *       # 2:00 AM daily
*/30 * * * *    # Every 30 minutes
0 */6 * * *     # Every 6 hours
0 0 * * 0       # Midnight Sunday
0 9 * * 1-5     # 9 AM, Monday-Friday
```

View your cron jobs:

```bash
crontab -l
```

Remove a job:

```bash
crontab -e  # Edit and delete the line
```

## Hands-On Lab

### Lab 5.1: Variables and Conditionals

Create `greet.sh`:

```bash
#!/bin/bash

# Get input from command line
name="$1"
hour=$(date +%H)

# Conditional greeting
if [ -z "$name" ]; then
    echo "Please provide a name: greet.sh Alice"
    exit 1
fi

if [ $hour -lt 12 ]; then
    echo "Good morning, $name"
elif [ $hour -lt 18 ]; then
    echo "Good afternoon, $name"
else
    echo "Good evening, $name"
fi
```

Run:

```bash
chmod +x greet.sh
./greet.sh Alice
# Output: Good morning/afternoon/evening, Alice (depending on time)
```

### Lab 5.2: Loops and File Processing

Create `process_logs.sh`:

```bash
#!/bin/bash

log_dir="/var/log"

echo "Processing log files in $log_dir..."

for logfile in $log_dir/*.log; do
    if [ ! -f "$logfile" ]; then
        continue
    fi
    
    lines=$(wc -l < "$logfile")
    size=$(du -h "$logfile" | cut -f1)
    
    echo "$logfile: $lines lines, $size size"
done
```

Run:

```bash
chmod +x process_logs.sh
./process_logs.sh
```

### Lab 5.3: Error Handling

Create `backup.sh`:

```bash
#!/bin/bash
set -e

SOURCE_DIR="/home/alice/documents"
BACKUP_DIR="/tmp/backups"

echo "Creating backup..."

# Check if source exists
if [ ! -d "$SOURCE_DIR" ]; then
    echo "Error: Source directory not found"
    exit 1
fi

# Create backup
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S).tar.gz" "$SOURCE_DIR"

echo "✓ Backup complete"
```

Run:

```bash
chmod +x backup.sh
./backup.sh
ls -lh /tmp/backups/
```

### Lab 5.4: Reading .env File

Create `.env`:

```
API_URL=http://localhost:8000
DB_HOST=localhost
DB_PORT=5432
DEBUG=true
```

Create `config.sh`:

```bash
#!/bin/bash

# Load environment
source .env

echo "Configuration:"
echo "  API URL: $API_URL"
echo "  DB Host: $DB_HOST"
echo "  DB Port: $DB_PORT"
echo "  Debug: $DEBUG"

# Use in commands
curl "$API_URL/health"
```

Run:

```bash
chmod +x config.sh
./config.sh
```

### Lab 5.5: Cron Scheduling

Check current cron jobs:

```bash
crontab -l
```

Edit to add a new job:

```bash
crontab -e
```

Add:

```
# Run a test script every minute
* * * * * /home/alice/test.sh >> /tmp/cron.log 2>&1
```

Verify:

```bash
sleep 65
cat /tmp/cron.log
```

Remove the job after testing:

```bash
crontab -e  # Delete the line
```

## Cheat Sheet: Bash Scripting Reference

### Variables and I/O

```bash
name="value"
echo $name
read input
env | grep VARIABLE
```

### Conditionals

```bash
if [ condition ]; then
    echo "yes"
else
    echo "no"
fi

[ -f file ]       # File exists
[ -d dir ]        # Directory exists
[ -z "$var" ]     # Empty string
[ "$var" = "val"] # String equals
[ $a -gt $b ]     # Greater than
```

### Loops

```bash
while [ $count -lt 10 ]; do
    count=$((count + 1))
done

for item in list; do
    echo $item
done

for i in {1..10}; do
    echo $i
done
```

### Functions

```bash
func_name() {
    echo "Hello, $1"
}

func_name "Alice"
result=$(func_name "Alice")
```

### Error Handling

```bash
set -e              # Exit on error
set -u              # Exit on undefined variable
set -x              # Debug (print commands)
set -o pipefail     # Error if any command in pipe fails

if [ $? -eq 0 ]; then
    echo "Success"
fi
```

### Cron Schedule Reference

```
0 0 * * *       # Daily midnight
0 2 * * *       # Daily 2 AM
*/5 * * * *     # Every 5 minutes
0 */6 * * *     # Every 6 hours
0 0 * * 0       # Weekly Sunday
0 0 1 * *       # Monthly
```

## Key Takeaways

- **Variables store data**, expand with `$name`
- **Conditionals branch logic**, use `[ condition ]` to test
- **Loops repeat actions**, `for` and `while` are most common
- **Functions modularize code**, call with `func_name arg1 arg2`
- **Exit codes matter**: `0` = success, non-zero = failure
- **set -e** exits on first error (essential for scripting)
- **source .env** loads environment variables
- **cron** schedules scripts to run automatically
- **Error handling** prevents silent failures in production

---

You've now mastered both Docker and Linux fundamentals. Combined, they form the foundation of modern backend engineering. The remaining 4 prompts will build specialized knowledge in databases, microservices architecture, testing, and deployment patterns.
