# Module 2: Processes and Services

## The Analogy: The Restaurant Kitchen

A restaurant kitchen has many operations running simultaneously:

- **Prep cooks** preparing ingredients
- **Line cooks** cooking main courses
- **Pastry chefs** making desserts
- **Dishwashers** cycling through plates

Each is a **process**—doing specific work. The **head chef** (the kernel) manages them all and ensures they don't interfere.

Linux works the same way. Your server is a bustling kitchen with dozens of processes running. You need to:

- See what's running (`ps`, `top`, `htop`)
- Stop processes that misbehave (`kill`)
- Start important things at boot (systemd services)
- Run long tasks in the background (`&`, `nohup`)

## What's a Process?

Every running program is a **process**. Each has:

- **PID** (Process ID) — Unique number (1-65535)
- **PPID** (Parent Process ID) — The process that started it
- **State** — Running (R), Sleeping (S), Stopped (T), Zombie (Z)
- **Resources** — CPU, memory, file handles

When you run `python app.py`, that's a process. When you run `docker run`, that's a process. When you open a file in vim, that's a process.

## Viewing Processes: ps, top, htop

### ps — Process Status

```bash
ps                      # Simple list (your terminal session)
ps aux                  # All processes, detailed
ps aux | grep python    # Grep for specific process
ps -eo pid,cmd,rss      # Custom columns (PID, command, memory)
```

Output of `ps aux`:

```
USER   PID   %CPU  %MEM    VSZ   RSS  TTY  STAT  START   TIME  COMMAND
alice  1234  0.7   2.3   156032 92184 pts/0 S    10:30  0:05  python app.py
```

| Column | Meaning |
|--------|---------|
| USER | Who owns the process |
| PID | Process ID |
| %CPU | CPU usage (0-100%) |
| %MEM | Memory usage (% of total) |
| VSZ | Virtual memory (in KB) |
| RSS | Resident set (actual memory used) |
| TTY | Terminal (pts/0 = pseudo-terminal 0) |
| STAT | State (S=sleeping, R=running, Z=zombie) |
| COMMAND | What's running |

### top — Real-Time Monitor

```bash
top
```

Shows all processes and updates every second:

```
top - 10:30:45 up 5 days, 2:15,  2 users,  load average: 1.23, 1.45, 1.32
Processes: 156 total,   2 running, 154 sleeping

  PID  USER   PR  NI    VIRT    RES    SHR %CPU  %MEM     TIME CMD
 1234  alice  20   0  156032  92184  45012  0.7   2.3   0:05 python
 5678  root   20   0  512000 256000 128000  1.2   4.1   0:20 postgres
 9012  alice  20   0   98765  38456  12345  0.3   1.2   0:02 redis-server
```

Press `q` to quit, `space` to refresh immediately.

### htop — Better top

`htop` is an improved version of `top` (not always installed):

```bash
sudo apt install htop
htop
```

Much more user-friendly:
- Color-coded
- Easier to navigate with arrow keys
- Click-able (on some systems)
- Shows CPU and memory bars

Press `q` to quit.

### Finding a Specific Process

```bash
ps aux | grep python          # Find Python processes
ps aux | grep postgres        # Find PostgreSQL
pgrep python                  # Just PID numbers
pidof nginx                   # Just PID of first match
```

## Controlling Processes: kill, killall, kill -9

### kill — Graceful Termination

```bash
kill 1234               # Send SIGTERM to process 1234
kill -15 1234           # Same (15 = SIGTERM)
kill 1234 5678 9012     # Kill multiple processes
```

A **graceful kill** says "please shutdown." The process has ~10 seconds to clean up (close files, flush database, etc.) before being forcefully terminated.

### kill -9 — Force Kill

```bash
kill -9 1234            # Force kill (SIGKILL)
pkill -9 python         # Force kill all Python processes
```

**Force kill** is violent. The process doesn't get to clean up. Use only when necessary (when normal kill doesn't work).

### killall — Kill by Name

```bash
killall python          # Kill all processes named "python"
killall -9 postgres     # Force kill all PostgreSQL
```

### Signals

| Signal | Number | Meaning | Graceful? |
|--------|--------|---------|-----------|
| SIGTERM | 15 | Terminate (default) | Yes |
| SIGKILL | 9 | Kill (forced) | No |
| SIGSTOP | 19 | Pause | N/A |
| SIGCONT | 18 | Resume | N/A |
| SIGHUP | 1 | Hangup (reload config) | N/A |

```bash
kill -SIGTERM 1234     # Graceful shutdown
kill -SIGKILL 1234     # Force kill
kill -SIGHUP 1234      # Tell nginx to reload config
```

## Background Jobs: &, nohup, screen, tmux

### Running in Background with &

```bash
python app.py &                 # Start in background
python app.py > app.log 2>&1 &  # Run and redirect output
```

The `&` puts the process in the background. You get your terminal prompt back immediately.

```bash
jobs                            # List background jobs
fg                              # Bring last job to foreground
bg                              # Resume background job that was stopped
```

### nohup — Immune to Hangup

When you close your SSH session, all child processes get SIGHUP (hangup) and terminate. `nohup` prevents this:

```bash
nohup python app.py > app.log 2>&1 &
```

Now if you disconnect SSH, the app keeps running.

### screen — Session Manager

`screen` creates a persistent terminal session:

```bash
screen -S myapp             # Create a new screen session named "myapp"
# Now you're in a new terminal
python app.py               # Run your app
# Press Ctrl+A then D to detach (app keeps running)

screen -ls                  # List sessions
screen -r myapp             # Reattach to "myapp"
```

### tmux — Better Session Manager

`tmux` is a modern alternative to screen:

```bash
tmux new-session -s myapp       # Start a session
# Or: tmux new -s myapp

# Inside tmux
Ctrl+B D                        # Detach
Ctrl+B C                        # Create new window
Ctrl+B N                        # Next window
Ctrl+B P                        # Previous window

tmux list-sessions              # List active sessions
tmux attach -t myapp            # Attach to session
```

## systemd Services: Managing System Daemons

**systemd** is the modern Linux init system that manages services.

### systemctl Commands

```bash
sudo systemctl start nginx          # Start service
sudo systemctl stop nginx           # Stop service
sudo systemctl restart nginx        # Restart service
sudo systemctl status nginx         # Check status
sudo systemctl enable nginx         # Start on boot
sudo systemctl disable nginx        # Don't start on boot
sudo systemctl is-active nginx      # Check if running
```

### Viewing Service Status

```bash
sudo systemctl status postgresql
```

Output:

```
● postgresql.service - PostgreSQL RDBMS
   Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
   Active: active (exited) since Mon 2024-01-15 10:30:45 UTC; 5 days ago
   Process: 1234 ExecStart=/usr/lib/postgresql/start (code=exited, status=0/SUCCESS)
  Main PID: 5678 (postgres)
     Tasks: 42
    Memory: 256M
```

### Service Files

System services are defined in `/etc/systemd/system/`. Example:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Python App
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload   # Load new service file
sudo systemctl start myapp
sudo systemctl enable myapp    # Start on boot
```

## Hands-On Lab

### Lab 2.1: View and Search Processes

```bash
# See all processes
ps aux | head -20

# Find Python processes
ps aux | grep python

# See details of PID 1
ps -eo pid,ppid,cmd | grep "^ *1 "

# Use pgrep
pgrep bash
```

### Lab 2.2: Start and Kill a Background Process

```bash
# Start a long-running task
python -c "import time; time.sleep(3600)" &
# Note the PID (or use jobs)

jobs
# Output: [1]+ Running ...

# Find its PID
ps aux | grep time.sleep

# Kill it gracefully
kill PIDNUMBER

# Or kill by name
killall python

# Force kill
kill -9 PIDNUMBER
```

### Lab 2.3: Run with nohup

```bash
# Create a simple script
cat > myapp.sh << 'EOF'
#!/bin/bash
echo "Starting app"
python -c "import time; time.sleep(3600)"
echo "App stopped"
EOF

chmod +x myapp.sh

# Run with nohup
nohup ./myapp.sh > app.log 2>&1 &

# Check the log
tail -f app.log

# Kill the background job
pkill -f myapp.sh
```

### Lab 2.4: Monitor with top/htop

```bash
# Use top
top
# Press: M (sort by memory), P (sort by CPU), q (quit)

# Or htop
htop
# Much nicer interface
```

### Lab 2.5: Check systemd Services

```bash
# List all services
systemctl list-units --type=service

# Check specific service
sudo systemctl status ssh
sudo systemctl status postgresql

# See if something is running
systemctl is-active ssh
```

## Cheat Sheet: Process Management Commands

| Command | Purpose |
|---------|---------|
| `ps aux` | List all processes |
| `top` | Real-time monitor |
| `htop` | Better real-time monitor |
| `kill PID` | Graceful termination |
| `kill -9 PID` | Force kill |
| `killall NAME` | Kill by process name |
| `pkill PATTERN` | Kill by pattern |
| `jobs` | List background jobs |
| `fg` | Bring to foreground |
| `bg` | Resume background |
| `nohup COMMAND &` | Immune to disconnect |
| `screen -S NAME` | New screen session |
| `screen -r NAME` | Attach to session |
| `tmux new -s NAME` | New tmux session |
| `tmux attach -t NAME` | Attach to session |
| `systemctl start SVC` | Start service |
| `systemctl stop SVC` | Stop service |
| `systemctl status SVC` | Check service |
| `systemctl enable SVC` | Start on boot |
| `ps -C NAME` | Processes by name |

## Key Takeaways

- **PID is the process identifier**, use it to control processes
- **SIGTERM is graceful** (kill), **SIGKILL is forced** (kill -9)
- **Background processes** keep running when you close the terminal if you use `nohup` or `screen`
- **systemd manages system services** with start/stop/enable/restart
- **top/htop show real-time CPU and memory** usage
- **Use kill sparingly**, preferably from the app itself

Now that you can manage processes, Module 3 covers permissions and users—critical for security.
