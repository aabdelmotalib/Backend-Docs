# Command Line Basics — Master the Terminal

The command line (terminal, shell, console) is your primary tool as a backend engineer.

You'll spend more time here than anywhere else.

---

## 1. What is the Command Line?

The **command line** is a text-based interface to your computer.

Instead of clicking buttons, you **type commands**.

### Terminal vs Command Prompt

| OS | Name | How to Open |
|----|------|-----------|
| **Linux** | Terminal | Ctrl + Alt + T |
| **macOS** | Terminal | Cmd + Space, type "terminal" |
| **Windows** | PowerShell | Win + R, type "powershell" |
| **Windows (legacy)** | Command Prompt | Win + R, type "cmd" |

**For this curriculum, all examples assume Linux/macOS terminal. Windows users should use PowerShell (it's similar).**

---

## 2. Your First Commands

### pwd — Print Working Directory

Shows you where you are right now.

```bash
$ pwd
/home/alice
```

**Translation**: I'm in `/home/alice` directory.

### ls — List

Shows files and folders in the current directory.

```bash
$ ls
Desktop    Documents    Downloads    Pictures
```

**With more details:**

```bash
$ ls -la
drwxr-xr-x  8 alice staff    256 Mar 12 10:30 .
drwxr-xr-x  5 root  wheel    160 Mar 12 09:15 ..
-rw-r--r--  1 alice staff    123 Mar 12 10:20 notes.txt
drwxr-xr-x  3 alice staff    96  Mar 12 10:15 projects
```

**-l** = long format (show more details)
**-a** = all (show hidden files that start with .)

### cd — Change Directory

Move to a different folder.

```bash
$ cd Desktop
$ pwd
/home/alice/Desktop
```

Go back up one level:

```bash
$ cd ..
$ pwd
/home/alice
```

Go to home directory:

```bash
$ cd ~
$ pwd
/home/alice
```

Go to root (top of the filesystem):

```bash
$ cd /
$ pwd
/
```

---

## 3. File and Folder Management

### mkdir — Make Directory

Create a new folder.

```bash
$ mkdir my_project
$ ls
my_project
```

### touch — Create File

Create an empty file.

```bash
$ touch notes.txt
$ ls
notes.txt
```

### cat — View File Contents

Display what's in a file.

```bash
$ cat notes.txt
This is my note
It has multiple lines
```

### cp — Copy

Copy a file.

```bash
$ cp notes.txt notes_backup.txt
$ ls
notes.txt    notes_backup.txt
```

Copy a folder (with everything inside):

```bash
$ cp -r my_project my_project_backup
```

**-r** = recursive (copy the folder and all its contents)

### mv — Move or Rename

Move a file to another folder:

```bash
$ mv notes.txt Desktop/
$ ls
Desktop
$ ls Desktop/
notes.txt
```

Rename a file:

```bash
$ mv notes.txt important_notes.txt
$ ls
important_notes.txt
```

### rm — Remove

Delete a file:

```bash
$ rm notes.txt
$ ls
(notes.txt is gone)
```

**⚠️ WARNING: There's no trash! Once deleted, it's gone forever. Use carefully.**

Delete a folder and everything inside:

```bash
$ rm -r my_project
```

**-r** = recursive (delete the folder and all its contents)

---

## 4. File Paths

### Absolute Paths

Start from the root `/`.

```bash
/home/alice/Desktop/notes.txt
/usr/bin/python3
/etc/config.txt
```

**Always work the same no matter where you are.**

### Relative Paths

Start from where you are now.

```
$ pwd
/home/alice

$ ls Desktop/notes.txt  # Relative path
$ ls /home/alice/Desktop/notes.txt  # Absolute path (same file)
```

### Special Paths

```
.           # Current directory
..          # Parent directory (one level up)
~           # Home directory (/home/alice)
-           # Previous directory
```

**Examples:**

```bash
$ cd .           # Stay in current directory (does nothing)
$ cd ..          # Go up one level
$ cd ../..       # Go up two levels
$ cd ~           # Go to home
$ cd ~/Desktop   # Home + Desktop
$ cd -           # Go to previous directory
```

---

## 5. Viewing and Editing Files

### cat — View Entire File

```bash
$ cat config.txt
name=Alice
age=25
city=New York
```

### less — View Large Files

```bash
$ less large_file.txt
(shows one page at a time)

# Navigate:
# Space = next page
# b = previous page
# q = quit
```

### head — View First Lines

```bash
$ head -5 data.txt
(shows first 5 lines)
```

### tail — View Last Lines

```bash
$ tail -10 data.txt
(shows last 10 lines)
```

### echo — Print Text

```bash
$ echo "Hello World"
Hello World

$ echo "Hello" > file.txt  # Write to file (replace)
$ echo "World" >> file.txt # Append to file
```

---

## 6. GREP — Search for Text

**GREP** = Global Regular Expression Print

Find text inside files.

### Simple Search

```bash
$ grep "Alice" users.txt
Alice,25,New York
Alice,30,Boston
(shows all lines with "Alice")
```

### Case-Insensitive Search

```bash
$ grep -i "alice" users.txt
alice,25,New York
ALICE,45,LA
Alice,30,Boston
```

**-i** = ignore case

### Count Matches

```bash
$ grep -c "Alice" users.txt
3
(3 lines contain "Alice")
```

**-c** = count

### Show Line Numbers

```bash
$ grep -n "Alice" users.txt
1:Alice,25,New York
3:Alice,30,Boston
8:Alice,45,LA
```

**-n** = line numbers

### Search Multiple Files

```bash
$ grep "error" *.log
app.log:ERROR: Connection failed
system.log:ERROR: Out of memory
(search all .log files)
```

---

## 7. Piping and Redirection

### Redirect Output to File

```bash
$ ls > file_list.txt
$ cat file_list.txt
Desktop
Documents
Downloads
```

**>** = write (overwrite file)
**>>** = append (add to file)

### Pipe — Use Output as Input

```bash
$ cat users.txt | grep "Alice"
Alice,25,New York
Alice,30,Boston
```

**|** = pipe (send output to next command)

### Combine Pipes

```bash
$ cat users.txt | grep "Alice" | wc -l
2
(count how many lines have "Alice")
```

**wc -l** = word count, lines

---

## 8. Running Programs

### Python

```bash
$ python3 my_script.py
(runs the Python script)
```

### Node.js

```bash
$ node my_script.js
(runs the JavaScript/Node script)
```

### With Arguments

```bash
$ python3 my_script.py arg1 arg2
(pass arguments to the program)
```

### Run in Background

```bash
$ python3 web_server.py &
(run in background)

$ jobs
(list background jobs)

$ fg
(bring to foreground)
```

---

## 9. Package Managers

### apt (Linux)

Install a package:

```bash
$ sudo apt update
$ sudo apt install python3-pip
```

**sudo** = run as admin (System User DO)

### homebrew (macOS)

```bash
$ brew install python@3.11
$ brew install node
```

### pip (Python Packages)

```bash
$ pip install requests
$ pip install flask
$ pip install django
```

### npm (Node.js Packages)

```bash
$ npm install express
$ npm install react
```

---

## 10. Useful Shortcuts

| Shortcut | What it does |
|----------|------------|
| **Ctrl + C** | Stop the running program |
| **Ctrl + Z** | Pause the program |
| **Ctrl + L** | Clear the screen |
| **Ctrl + A** | Go to beginning of line |
| **Ctrl + E** | Go to end of line |
| **↑** | Previous command |
| **↓** | Next command |
| **Tab** | Auto-complete (press twice for options) |

---

## 11. Viewing Your Command History

### history

```bash
$ history
1  pwd
2  ls -la
3  mkdir my_project
4  cd my_project
5  echo "hello" > file.txt
```

### Ctrl + R

Search in your history:

```bash
$ Ctrl + R
(reverse-i-search)`': 
```

Type part of a command to find it:

```bash
(reverse-i-search)`cd': cd my_project
```

Press Enter to run it, or Ctrl + C to cancel.

---

## 12. Exit Codes

When a program finishes, it returns an **exit code**.

- **0** = Success ✅
- **Non-zero** = Error ❌

### Check Exit Code

```bash
$ ls
Desktop Documents

$ echo $?
0
(Success!)

$ ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory

$ echo $?
2
(Error! Code 2)
```

---

## 13. Hands-On: Complete Workflow

Let's create a project and organize files:

```bash
# Create project directory
$ mkdir my_backend
$ cd my_backend

# Create subdirectories
$ mkdir src data logs

# Create files
$ touch src/main.py
$ touch README.md
$ touch requirements.txt

# Check structure
$ ls -la
drwxr-xr-x  5 alice staff   160 Mar 12 10:30 src
drwxr-xr-x  5 alice staff   160 Mar 12 10:30 data
drwxr-xr-x  5 alice staff   160 Mar 12 10:30 logs
-rw-r--r--  1 alice staff     0 Mar 12 10:30 README.md
-rw-r--r--  1 alice staff     0 Mar 12 10:30 requirements.txt

# Write to a file
$ echo "My Backend Project" > README.md
$ cat README.md
My Backend Project

# Copy project
$ cd ..
$ cp -r my_backend my_backend_backup

# Clean up test
$ rm -r my_backend_backup
```

---

## 14. Cheat Sheet

| Command | Purpose | Example |
|---------|---------|---------|
| **pwd** | Print working directory | `pwd` |
| **ls** | List files | `ls` or `ls -la` |
| **cd** | Change directory | `cd Desktop` |
| **mkdir** | Make directory | `mkdir my_project` |
| **touch** | Create file | `touch notes.txt` |
| **cat** | View file | `cat notes.txt` |
| **cp** | Copy | `cp file.txt file2.txt` |
| **mv** | Move/rename | `mv old.txt new.txt` |
| **rm** | Delete | `rm file.txt` |
| **grep** | Search text | `grep "text" file.txt` |
| **echo** | Print text | `echo "hello"` |
| **pipe** | Use output as input | `cat file \| grep "text"` |
| **>** | Redirect to file | `ls > list.txt` |
| **>>** | Append to file | `echo "x" >> file.txt` |
| **sudo** | Admin privileges | `sudo apt install` |
| **history** | Command history | `history` |

---

## Next Steps

1. **Practice**: Open a terminal and run these commands
2. **Explore**: Navigate your system, create files, use grep
3. **Memorize**: The shortcuts and common commands
4. **Understand**: When to use pipes, redirection, and searching

---

## You're Ready!

You now understand:

✅ How to navigate the file system

✅ How to create, view, and delete files

✅ How to search for text with grep

✅ How to run programs

✅ How to pipe and redirect

On to the main curriculum! You'll use these commands constantly.

→ **[Back to Prerequisites](index.md)** or **[Docker & Containers](../docker/index.md)**
