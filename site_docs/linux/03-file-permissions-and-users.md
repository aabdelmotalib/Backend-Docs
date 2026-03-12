# Module 3: File Permissions and Users

## The Analogy: Apartment Door Locks

Your apartment has a door with different types of access:

- **You (owner)**: Full access. You can enter, modify, invite people, change the locks
- **Roommate (group)**: Limited access. They can enter and use the kitchen, but not your private room
- **Guest (others)**: Very limited. They can enter common areas, but not your bedroom
- **Everyone (world)**: No access. They can't enter at all

Linux file permissions work exactly this way.

## File Permissions Basics

Every file has three types of permissions for three categories:

### The Three Categories

1. **Owner (User)** — The person who created the file
2. **Group** — A group of users
3. **Others (World)** — Everyone else

### The Three Permissions

1. **Read (r)** — View the file contents
2. **Write (w)** — Edit/delete the file
3. **Execute (x)** — Run the file (if it's a script or binary)

## Reading Permissions: The rwx System

```bash
ls -l /home/alice/notes.txt
# Output: -rw-r--r-- 1 alice alice 1024 Jan 15 10:30 notes.txt
```

Breaking it down:

```
-rw-r--r--
│ │││ │││
│ │││ └──┬── Others: r-- (read only)
│ └──┬─── Group: r-- (read only)
├─ Owner: rw- (read, write)
└─ File type: - (- = regular file, d = directory, l = symlink)
```

So:
- **Owner (alice)**: Can read and write (rw-)
- **Group (alice)**: Can only read (r--)
- **Others**: Can only read (r--)

### Common Permission Patterns

```
-rw-r--r--    # File: Owner rw, Group r, Others r (typical)
-rw-------    # File: Owner rw, Nobody else reads (private)
drwxr-xr-x    # Directory: Owner rwx, Group rx, Others rx
-rwxr-xr-x    # Executable: Owner rwx, Group rx, Others rx
-rw-rw-rw-    # File: Everyone can read and write (dangerous!)
-r--------    # File: Only owner can read (very private)
```

## chmod — Change Permissions

### Text Notation

```bash
chmod u+x script.sh          # User: add execute
chmod g+r file.txt           # Group: add read
chmod o-r file.txt           # Others: remove read
chmod a+r file.txt           # All: add read
chmod u+w,g-w file.txt       # Multiple changes
chmod g=r file.txt           # Group: set to read only (clear others)
```

Letters:
- `u` = user (owner)
- `g` = group
- `o` = others
- `a` = all

Operations:
- `+` = add permission
- `-` = remove permission
- `=` = set exactly

### Numeric Notation

File permissions can also be represented as three numbers:

```
r w x
4 2 1
```

Each digit is a sum:

- `7` = 4+2+1 = rwx (full permissions)
- `6` = 4+2 = rw- (read, write, no execute)
- `5` = 4+1 = r-x (read, execute, no write)
- `4` = r-- (read only)
- `0` = --- (no permissions)

So `chmod 755 script.sh` means:

```
7     5     5
rwx   r-x   r-x
Owner Group Others
```

**Common numeric patterns:**

| Permissions | Use Case |
|-----------|----------|
| `755` | Executable: `rwxr-xr-x` |
| `644` | File: `rw-r--r--` |
| `600` | Private: `rw-------` |
| `700` | Private dir: `rwx------` |
| `777` | Everyone can do anything (dangerous!) |

## chown — Change Owner and Group

```bash
chown alice file.txt              # Change owner to alice
chown alice:developers file.txt   # Change owner to alice, group to developers
chown -R alice directory/         # Recursive (all files in directory)
```

You can only do this as root or with `sudo`.

## Users and Groups

### Who am I?

```bash
whoami                   # Current user
id                       # User info (UID, groups)
groups                   # Groups you belong to
```

Output:

```
$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),4(adm),24(cdrom)
```

### Managing Users

```bash
sudo useradd appuser                    # Create user
sudo userdel appuser                    # Delete user
sudo passwd appuser                     # Set password
sudo usermod -aG groupname appuser      # Add to group
```

### /etc/passwd — User Database

Every user on the system is listed here:

```bash
cat /etc/passwd
```

Output:

```
root:x:0:0:root:/root:/bin/bash
alice:x:1000:1000:Alice User:/home/alice:/bin/bash
appuser:x:1001:1001::/home/appuser:/usr/sbin/nologin
```

Format: `username:password_hash:UID:GID:fullname:home:shell`

## The Critical Docker Security Rule: Don't Run as Root

By default, Docker containers run as **root**. This is dangerous:

- If someone breaks into your container, they have root
- The container can access host resources with privilege
- There's no containment—it's root inside and might compromise the host

### Creating a Non-Root User in Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Create a non-root user
RUN useradd -m -u 1000 appuser

# Copy code
COPY --chown=appuser:appuser app.py .

# Install dependencies
RUN pip install --no-cache-dir fastapi uvicorn

# Switch to non-root user BEFORE running the app
USER appuser

CMD ["uvicorn", "app:app", "--host", "0.0.0.0"]
```

**Why `-m -u 1000`?**

- `-m`: Create home directory
- `-u 1000`: Assign UID 1000 (standard for first non-root user)

### Verifying Non-Root User

```bash
docker run myapp whoami
# Output: appuser (not root!)

docker run myapp id
# Output: uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

## Directory Permissions: Read, Write, Execute

For directories, the permissions mean:

- **r (read)**: List contents (`ls`)
- **w (write)**: Create/delete files inside
- **x (execute)**: Enter the directory (`cd`)

```bash
chmod 755 /var/www/html    # Owner rwx, Group rx, Others rx
chmod 700 /home/alice      # Owner rwx only
```

To enter a directory, you **must** have execute permission on it.

## Hands-On Lab

### Lab 3.1: Understanding Permissions

```bash
# Create test files
mkdir /tmp/perms_test
cd /tmp/perms_test

echo "secret data" > secret.txt
echo "public data" > public.txt

# View permissions
ls -l

# Change public.txt to read-only for everyone
chmod 444 public.txt
ls -l public.txt

# Try to edit it
echo "test" > public.txt  # Should fail: Permission denied

# Restore write permission for owner
chmod u+w public.txt
echo "test" > public.txt  # Now it works!

# Make secret.txt private (owner only)
chmod 600 secret.txt
ls -l secret.txt

# Try to read as another user
sudo -u nobody cat secret.txt  # Should fail: Permission denied
```

### Lab 3.2: Numeric Permissions

```bash
cd /tmp/perms_test

# Create a script
cat > hello.sh << 'EOF'
#!/bin/bash
echo "Hello"
EOF

# Try to run it
./hello.sh  # Fails: Permission denied

# Add execute permission (755)
chmod 755 hello.sh

# Now it works
./hello.sh  # Output: Hello

# Check permissions
ls -l hello.sh
# Should show: -rwxr-xr-x
```

### Lab 3.3: Create a Non-Root User

```bash
# Create a user
sudo useradd -m -s /bin/bash testuser

# Set a password (optional for this test)
sudo passwd testuser

# Switch to that user
sudo -u testuser whoami

# Check what groups testuser belongs to
id testuser

# Add testuser to sudo group
sudo usermod -aG sudo testuser

# Verify
groups testuser
```

### Lab 3.4: File Ownership

```bash
# Create a file as root
sudo touch /tmp/root_file.txt

# View owner
ls -l /tmp/root_file.txt
# Should show: -rw-r--r-- 1 root root

# Change owner to testuser
sudo chown testuser /tmp/root_file.txt

# Verify
ls -l /tmp/root_file.txt
# Should show: -rw-r--r-- 1 testuser users

# testuser can now edit it
sudo -u testuser echo "Hello" > /tmp/root_file.txt
```

### Lab 3.5: Docker Container Permissions

Create this Dockerfile:

```dockerfile
FROM python:3.12-slim

RUN useradd -m appuser

COPY app.py .

USER appuser

CMD ["python", "app.py"]
```

Create app.py:

```python
import os
print(f"Running as: {os.getuid()}")
```

Build and run:

```bash
docker build -t perm_test .

docker run perm_test
# Output: Running as: 1000 (not 0/root)
```

## Cheat Sheet: chmod Numeric Reference

| Permissions | Octal | Use |
|-----------|-------|-----|
| rwx------ | 700 | Private directory |
| rwxr-xr-x | 755 | Executable/directory (public) |
| rw-r--r-- | 644 | File (readable) |
| rw------- | 600 | Private file |
| r-------- | 400 | Read-only (private) |
| rwxrwxrwx | 777 | Everyone can do anything (AVOID) |

### Common chmod Operations

```bash
chmod 755 script.sh      # Make executable
chmod 644 file.txt       # Make readable
chmod 600 .ssh/id_rsa    # Private key (very secure)
chmod -R 755 /var/www    # Recursive
chmod u+x script.sh      # User add execute
chmod g-w file.txt       # Group remove write
chmod o-r secret.txt     # Others remove read
chmod a+r file.txt       # All add read
```

## Key Takeaways

- **rwx = read, write, execute**, each worth 4, 2, 1 points
- **Three categories: owner (u), group (g), others (o)**
- **chmod changes permissions**, chown changes owner
- **755 = executable** (rwxr-xr-x), **644 = readable** (rw-r--r--)
- **Never run containers as root** — create a non-root user
- **Private keys should be 600** (owner read/write only)
- **Directories need execute permission to enter** (`cd` requires `x`)

Now that you can manage permissions and users, Module 4 covers networking commands for API testing and debugging.
