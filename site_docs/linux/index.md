# Linux Fundamentals for Backend Engineers

Welcome to the Linux section of this comprehensive backend engineering guide. You might be running Docker on macOS or Windows, but inside every container is Linux. Understanding Linux isn't optional—it's foundational.

## Why Every Backend Engineer Needs Linux Mastery

When your app crashes in production, you don't SSH into a GUI and click things. You:

1. SSH into the server
2. Use command-line tools to diagnose the issue
3. Fix it with scripts and configuration files
4. Never see a mouse or desktop

Ninety-nine percent of backend infrastructure runs on Linux. You'll interact with:

- **Servers** (AWS EC2, DigitalOcean, Linode all use Linux)
- **Containers** (Every Docker container is Linux-based)
- **Databases** (PostgreSQL, MySQL, MongoDB run on Linux)
- **Message Queues** (Redis, RabbitMQ run on Linux)
- **CI/CD Systems** (GitHub Actions, GitLab CI run jobs on Linux servers)

If you don't understand Linux, you're flying blind in production.

## What You'll Learn

This section covers Linux fundamentals in 5 modules, designed specifically for backend engineers:

1. **Linux Essentials** — File system, navigation, basic commands
2. **Processes and Services** — Understanding what's running, managing services with systemd
3. **File Permissions and Users** — Security, ownership, Docker user best practices
4. **Networking Commands** — Tools for API testing, debugging connections
5. **Bash Scripting for DevOps** — Writing deployment scripts, automation

## Module Dependencies

```
01 → 02 → 03 ↘
            04 ↘
              05
```

- **Module 1** is the foundation (navigation, commands)
- **Module 2** builds on it (managing processes)
- **Module 3** adds security (permissions, users)
- **Modules 4-5** use everything for real-world tasks

## The Linux You'll Use

Throughout this guide, I reference **Ubuntu 22.04 LTS** (the long-term support version used in most production servers). However, everything applies to other Linux distributions:

- **CentOS/RHEL** — Enterprise servers (uses `yum` instead of `apt`)
- **Debian** — Similar to Ubuntu (uses `apt`)
- **Alpine** — Minimal, used in Docker images (uses `apk`)

The commands and concepts are the same. Only package managers differ slightly.

## Before You Start

You should have:

- **Access to a Linux machine** (or Docker container with bash)
- **Basic comfort with command line** (navigating folders, running commands)
- **A text editor** (vim, nano, or any editor)

All examples assume you're in a bash shell on Ubuntu 24.04.

Now let's start with Module 1 and understand the Linux file system.
