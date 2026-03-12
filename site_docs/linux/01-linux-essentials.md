# Module 1: Linux Essentials

## The Analogy: The Library

The Linux file system is like a library:

- **Your home directory** (`/home/yourname`) is your personal workspace
- **Root directory** (`/`) is the main library building
- **Folders** are sections (Linux, Science, Fiction)
- **Files** are the books
- **Shortcuts** (symlinks) let you reference books from multiple sections

The library has a catalog system (commands like `ls`, `find`) and checkout rules (permissions—covered in Module 3).

## The File System Structure

Everything in Linux starts from `/` (the root directory). Here's the key folders you'll interact with:

### Core Directories

```
/
├── /home/          # User home directories (/home/alice, /home/bob)
├── /root/          # Root user's home directory
├── /etc/           # Configuration files (nginx.conf, postgresql.conf, etc.)
├── /var/           # Variable data (logs, databases, temporary files)
├── /tmp/           # Temporary files (deleted on reboot)
├── /usr/           # User programs (Python, Ruby, Node.js, etc.)
├── /usr/bin/       # Executable programs (/usr/bin/python, /usr/bin/curl, etc.)
├── /usr/local/     # Locally installed programs (from source)
└── /opt/           # Optional software packages
```

### Important for Backend Developers

```
/etc/               # Configuration
  /etc/nginx/       # Nginx config
  /etc/postgresql/  # PostgreSQL config
  /etc/docker/      # Docker daemon config

/var/               # Logs and data
  /var/log/         # Application logs
  /var/lib/         # Databases, file storage
  /var/lib/postgresql/  # PostgreSQL data

/tmp/               # Temporary files (cleaned on reboot)
  /tmp/cache/       # App caches

/home/              # User files
  /home/alice/      # Alice's home
  /home/alice/.ssh/ # SSH keys
```

!!! tip
    `/tmp` is always safe to use for temporary files. They're automatically deleted on reboot.

## Navigation: cd, ls, pwd

### pwd — Print Working Directory

Shows where you are:

```bash
pwd
# Output: /home/alice/projects/myapp
```

### ls — List Directory Contents

```bash
ls                          # List files in current dir
ls -l                       # Long format (permissions, size, date)
ls -la                      # Include hidden files (starting with .)
ls -lh                      # Human-readable sizes (5MB, 200KB, not bytes)
ls /var/log                 # List a specific directory
ls -S                       # Sort by size (largest first)
ls -t                       # Sort by modification time
```

Output example:

```
$ ls -l /home/alice
total 48
drwxr-xr-x  2 alice alice 4096 Jan 15 10:30 projects
-rw-r--r--  1 alice alice 1024 Jan 14 15:20 notes.txt
lrwxrwxrwx  1 alice alice    9 Jan 13 08:00 link_to_projects -> projects
```

What each column means:

| Column | Meaning |
|--------|---------|
| `drwxr-xr-x` | File permissions (d=directory) |
| `2` | Hard links count |
| `alice` | Owner |
| `alice` | Group |
| `4096` | Size in bytes |
| `Jan 15 10:30` | Last modified |
| `projects` | Filename |

### cd — Change Directory

```bash
cd /home/alice           # Absolute path
cd projects              # Relative path (from current dir)
cd ..                    # Parent directory
cd ~                     # Home directory
cd -                     # Previous directory
cd /                     # Root directory
```

### mkdir — Make Directory

```bash
mkdir projects
mkdir -p /path/to/deep/folders  # -p creates parent dirs if needed
```

## File Operations

### cp — Copy

```bash
cp file.txt copy.txt                    # Copy file
cp -r directory/ directory_backup/      # Copy directory recursively
cp *.log /tmp/                          # Copy all .log files to /tmp/
```

### mv — Move / Rename

```bash
mv old_name.txt new_name.txt           # Rename file
mv file.txt /tmp/                      # Move to another directory
mv /home/alice/file.txt /tmp/file.txt  # Move and rename
```

### rm — Remove

```bash
rm file.txt                  # Delete file
rm -r directory/            # Delete directory and contents
rm -f file.txt              # Force delete (no confirmation)
```

!!! danger
    `rm` is permanent. There's no trash/recycle bin. Be careful with `rm -r`.

### cat — Print File Contents

```bash
cat file.txt                  # Print entire file
cat file1.txt file2.txt       # Print multiple files
cat /var/log/syslog           # View system logs
```

### less — View File Paged

For large files, `less` shows one screen at a time:

```bash
less /var/log/syslog

# Inside less:
# Space    → Next page
# b        → Previous page
# g        → Go to top
# G        → Go to bottom
# /search  → Search for text
# q        → Quit
```

### tail — View End of File

```bash
tail /var/log/syslog           # Last 10 lines
tail -50 /var/log/syslog       # Last 50 lines
tail -f /var/log/syslog        # Follow (watch new lines appear)
```

The `-f` flag is super useful for watching logs in real-time.

## Searching for Content and Files

### grep — Search File Content

```bash
grep "error" syslog              # Find lines containing "error"
grep -i "error" syslog           # Case-insensitive
grep -n "error" syslog           # Show line numbers
grep -c "error" syslog           # Count matching lines
grep "error\|warning" syslog     # Multiple patterns
```

### find — Search for Files

```bash
find /var/log -name "*.log"                # Files ending with .log
find /var/log -name "syslog*"              # Files starting with syslog
find /var/log -type f -size +1M            # Files larger than 1MB
find /var/log -type f -mtime -7            # Modified in last 7 days
find /var/log -type d                      # Only directories
```

## Text Editing

### nano — Simple Editor

```bash
nano file.txt
# Edit the file
# Ctrl+O to save, Ctrl+X to quit
```

`nano` is beginner-friendly. You just type and press Ctrl+O to save.

### vim — Advanced Editor

```bash
vim file.txt
```

vim is powerful but steep learning curve:

```
i        → Enter insert mode (type)
Esc      → Exit insert mode
:w       → Save (write)
:q       → Quit
:wq or :x → Save and quit
dd       → Delete line
```

**Pro tip**: Use nano for quick edits, vim for complex editing when you're comfortable.

### echo — Print Text

```bash
echo "Hello World"                              # Print to terminal
echo "Hello" > file.txt                         # Write to file (overwrite)
echo "Hello" >> file.txt                        # Append to file
echo "DB_PASSWORD=secret" > .env                # Create .env file
```

## Working with Absolute vs Relative Paths

### Absolute Paths (Start with /)

From anywhere, they point to the same location:

```bash
/home/alice/projects/myapp/src/main.py
```

### Relative Paths (No leading /)

Depend on your current directory:

```bash
# If you're in /home/alice/projects
myapp/src/main.py          # Same as /home/alice/projects/myapp/src/main.py

# If you're in /home/alice/projects/myapp
./src/main.py              # Same location (. means current dir)
../projects/myapp/src/main.py  # Go up one directory
```

### . and ..

```bash
.               # Current directory
./script.sh     # Script in current directory
..              # Parent directory
../..           # Two levels up
```

## Package Management on Ubuntu

### apt — Package Manager

Ubuntu uses `apt` to install software:

```bash
sudo apt update                  # Update package list
sudo apt install package_name    # Install a package
sudo apt remove package_name     # Remove a package
sudo apt upgrade                 # Upgrade all packages
sudo apt autoremove              # Remove unused dependencies
```

Examples:

```bash
sudo apt update
sudo apt install curl            # Install curl (for API testing)
sudo apt install postgresql-client  # Install PostgreSQL client
sudo apt install python3-pip     # Install pip (Python package manager)
```

!!! note
    `sudo` means "superuser do"—run the command as root. You need elevated privileges to install software.

## Hands-On Lab

### Lab 1.1: Navigate and Explore

```bash
# See where you are
pwd

# Go to root
cd /

# List what's there
ls -l

# See /home
cd /home
ls -l

# Go to your home
cd ~
pwd

# List your home
ls -la
```

### Lab 1.2: Create, Edit, and View Files

```bash
# Go to /tmp
cd /tmp

# Create a directory
mkdir myproject
cd myproject

# Create a file with echo
echo "This is my first file" > readme.txt

# View it
cat readme.txt

# Edit it with nano
nano readme.txt
# Add another line, then Ctrl+O to save, Ctrl+X to quit

# View again
cat readme.txt

# Append more content
echo "Added another line" >> readme.txt
tail readme.txt  # View end of file

# Copy the file
cp readme.txt readme_backup.txt
ls -l
```

### Lab 1.3: Find Large Files

```bash
# Go to /var/log (system logs)
cd /var/log

# Find the 3 largest files
ls -lSh | head -5

# Or search for all log files
find . -name "*.log" -type f -size +1M

# Count matching files
find . -name "*.log" | wc -l
```

### Lab 1.4: Search Log Content

```bash
# View end of syslog
tail -20 /var/log/syslog

# Search for errors
grep -i "error" /var/log/syslog | head -5

# Count errors
grep -i "error" /var/log/syslog | wc -l
```

### Lab 1.5: Install Software

```bash
# Update package list (might need sudo)
sudo apt update

# Install curl (command-line HTTP client)
sudo apt install -y curl

# Verify it's installed
which curl
```

## Cheat Sheet: 30 Essential Linux Commands

| Command | Usage |
|---------|-------|
| `pwd` | Print working directory |
| `ls` | List files |
| `cd` | Change directory |
| `mkdir` | Create directory |
| `rmdir` | Remove empty directory |
| `rm` | Remove file |
| `cp` | Copy |
| `mv` | Move/rename |
| `cat` | Print file |
| `less` | View paged |
| `tail -f` | Follow log |
| `head` | View top lines |
| `grep` | Search content |
| `find` | Search files |
| `touch` | Create empty file |
| `echo` | Print text |
| `wc` | Count lines/words |
| `sort` | Sort lines |
| `uniq` | Remove duplicates |
| `cut` | Extract columns |
| `sed` | Edit stream |
| `awk` | Text processing |
| `chmod` | Change permissions |
| `chown` | Change owner |
| `sudo` | Superuser do |
| `apt install` | Install package |
| `apt remove` | Remove package |
| `man` | Manual (help) |
| `which` | Find command location |
| `whoami` | Current user |

**Pro tip**: `man COMMAND` shows the manual for any command. Example: `man ls` explains all `ls` flags.

## Key Takeaways

- **`/` is root**, everything else is inside it
- **`/home/` contains user directories**, `/root/` for root user
- **`/etc/` has configuration files**, `/var/logs/` has logs
- **`cd` navigates, `ls` lists, `cat` shows content**
- **Absolute paths start with `/`, relative paths don't**
- **`>` overwrites, `>>` appends** files
- **`sudo` elevates privilege** for installs
- **`grep` searches content, `find` searches filenames**

Now that you can navigate Linux, Module 2 teaches you to manage what's running on it.
