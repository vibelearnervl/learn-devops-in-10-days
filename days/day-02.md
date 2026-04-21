---
day: 02
title: "Linux & shell — the mental model, not the cheat sheet"
duration_min: 60
concepts: ["linux", "shell", "cli", "permissions"]
ai_prompts:
  - "Explain what happens between the moment I press Enter on a shell command and the moment I see output. Walk through every step."
  - "Describe how Linux file permissions work using an office building and keycards as an analogy."
  - "I'm debugging a slow server. Give me five Linux commands in priority order and explain what each one reveals."
shorts_ids: []
---

## The one idea for today

**Linux is not a pile of commands to memorise — it's a mental model.** Every DevOps tool you'll learn this week (Docker, Kubernetes, Terraform, CI runners) is ultimately a thin wrapper over Linux primitives: processes, files, users, networks. If you understand the primitives, the tools become obvious. If you don't, you'll forever feel like you're pattern-matching against Stack Overflow.

In the AI era, the skill isn't typing `awk` flags from memory. It's knowing *what question to ask*, then asking Claude to generate the exact command. Learn the model. Prompt the syntax.

## Core concepts

### Everything is a file
Linux treats devices, sockets, pipes, and even running processes as files. Your hard disk is `/dev/sda`. Your terminal is `/dev/tty`. Every running process exposes its state under `/proc/<pid>/`. This isn't poetry — it means one mental model (open, read, write, close) covers almost everything the OS does.

### Processes, PIDs, and the fork-exec model
A process is a running program. Every process has a PID (process ID) and a parent PID. When you run a command, your shell **forks** itself (makes a copy), then **execs** the new binary over that copy. This fork-then-exec dance is how every Linux program starts. When a parent dies before the child, the child becomes a "zombie" until something reaps it — a classic interview topic.

### File permissions as a 3×3 matrix
Every file has three classes of accessor — **user** (owner), **group**, **other** — and three rights for each: **read (r), write (w), execute (x)**. That's 9 bits, displayed as `rwxr-xr--`. Think of it as an office building: the user is your private office, the group is your floor, "other" is the lobby. `chmod 755 file` = owner gets full access, group/lobby can read and execute but not modify.

### The shell as an interpreter
The shell (bash, zsh, fish) is a program that reads your commands, parses them, expands variables and globs, then calls the appropriate binaries. Your `ls` isn't magic — it's a binary at `/usr/bin/ls` that the shell located via `$PATH` and executed. Understand this and `which`, `type`, and `env` stop being mysterious.

### Pipes and redirection — composition over tools
`ls | grep foo | wc -l` works because each program reads from **stdin** and writes to **stdout**. The pipe (`|`) wires one program's stdout to the next program's stdin. This composability is the Unix philosophy: *small tools that do one thing well, chained together*. AI can generate complex one-liners, but you need to understand pipes to read and debug them.

### Environment variables and PATH
Environment variables are key-value strings inherited by every child process. `$PATH` is special — it's a colon-separated list of directories the shell searches when resolving commands. Every DevOps tool you install (Docker, kubectl, terraform) drops a binary into one of these directories.

### Users, groups, and sudo
Linux is multi-user by design. Each user has an ID (UID). Groups bundle users with shared permissions. `sudo` lets a user run a command *as another user* (usually root), gated by the `/etc/sudoers` file. The golden rule: **give the least privilege that works.** Production services should never run as root.

### Systemd — how services live and die
Systemd is the init system on modern Linux. It manages **units** (services, timers, mount points). When you `systemctl start nginx`, systemd forks the nginx process, watches it, and restarts it if it dies. Almost every container orchestrator is a distributed version of what systemd does on one machine.

### Package managers
`apt` (Debian/Ubuntu), `yum`/`dnf` (RHEL/Fedora), `apk` (Alpine) — different names, same job. They fetch signed archives, resolve dependencies, and install files to well-known paths. Interview tip: understand that a package manager doesn't *compile* the software — it installs pre-built binaries.

### SSH and the mental model of asymmetric keys
SSH uses a key pair. Your **private key** stays on your laptop forever. The **public key** goes into `~/.ssh/authorized_keys` on the server. When you connect, the server sends a challenge that only your private key can sign. It's mathematical trust — no passwords travel over the wire.

### Networking at the Linux level
`ss -tlnp` shows listening TCP ports. `ip a` shows interfaces. `curl -v` traces an HTTP request. You don't need to memorise these — you need to know *what they reveal*. When a service "isn't reachable," you check: is it listening? is the port open? is DNS resolving?

## Common gotchas

- `chmod 777` "fixes" permissions but creates gaping security holes — never use it in production
- `rm -rf /` used to wipe your system; modern Linux has `--preserve-root` by default, but the lesson stands: destructive commands are irreversible
- `cron` jobs run in a stripped environment — `$PATH` is minimal, so always use absolute paths
- Editing files with `sudo vim` makes them owned by root, breaking the app that reads them
- Piping `cat` into `grep` is redundant — `grep foo file.txt` is cleaner than `cat file.txt | grep foo`
- `$HOME` for the root user is `/root`, not `/home/root` — a common script bug

## Interview questions with model answers

### Q: What happens when I type `ls` and press Enter?
**Short answer:** The shell parses the command, looks up `ls` in `$PATH`, forks a child process, exec's the `ls` binary into it, and waits for the child to exit before showing the prompt again.
**Deeper:** First the shell tokenises your input and expands any variables, aliases, or globs. It checks if `ls` is a builtin (it isn't), then searches each directory in `$PATH` until it finds `/usr/bin/ls`. It calls `fork()` to create a copy of itself, the child calls `execve()` to replace its memory with the `ls` binary, and the parent shell calls `wait()` on the child's PID. The child runs, writes directory entries to stdout, and exits. The parent reads the exit code, stores it in `$?`, and prints the prompt.
**Watch out for:** Candidates often forget the fork-exec distinction. They'll say "the shell runs ls" — true but shallow. Senior interviewers want the two-step model.

### Q: Explain Linux file permissions.
**Short answer:** Three permission classes (user, group, other), each with three rights (read, write, execute), giving nine bits shown as `rwxrwxrwx` or as three octal digits like `755`.
**Deeper:** Each file stores an owner UID and group GID. When a process tries to access the file, the kernel checks: does the process UID match the owner? If yes, use the user bits. Does the process's group list include the file's GID? If yes, use the group bits. Otherwise, use the other bits. `chmod` changes the bits; `chown` changes the owner/group. Directories use the same bits but with different semantics — execute on a directory means "can traverse into it."
**Watch out for:** Many candidates forget that execute-on-a-directory is a traversal bit, not an execution bit. Also, the setuid/setgid/sticky bits (the 4th octal digit) are classic follow-up territory.

### Q: What's the difference between a hard link and a symlink?
**Short answer:** A hard link is a second name for the same inode (the actual file). A symlink is a tiny file whose contents are a path to another file.
**Deeper:** Delete the original file and its hard links still work — the inode remains because reference count is positive. Delete the original and symlinks become dangling pointers. Hard links can't cross filesystems (inodes are filesystem-local); symlinks can. Hard links can't point to directories (prevents loops); symlinks can. Knowing this matters when debugging "file not found" errors in deploys that use symlinks.
**Watch out for:** Candidates confuse `ln` (hard link by default) with `ln -s` (symlink).

### Q: How does SSH authentication work without sending a password?
**Short answer:** The server has your public key. You have your private key. During the handshake, the server encrypts a random challenge with your public key, and only your private key can decrypt it — proving identity without sending the secret.
**Deeper:** Actually it's more nuanced: the client signs a session identifier with its private key, and the server verifies the signature against the authorised public key. The private key never leaves your machine and the password (if any) only unlocks the local private key file. This is why `~/.ssh/authorized_keys` on the server lists *public* keys, never private ones.
**Watch out for:** Candidates often say "the server decrypts with the public key." Public keys verify and encrypt; private keys sign and decrypt. Mixing these up is a red flag.

### Q: A user reports the server is "slow." How do you investigate?
**Short answer:** Use the USE method — check Utilization, Saturation, and Errors for CPU, memory, disk, and network.
**Deeper:** Start with `top` or `htop` to see CPU and memory pressure. `vmstat 1` shows if the system is swapping or blocked on I/O. `iostat -x 1` reveals disk queue depth. `ss -s` summarises socket states. `dmesg | tail` catches kernel-level issues (OOM kills, disk errors). You're not looking for the answer — you're narrowing the category. Once you know "it's disk-bound" or "CPU-bound," you have a direction.
**Watch out for:** Jumping straight to logs. Logs tell you what *happened* after you know *where* to look.

### Q: What's the difference between `sudo` and `su`?
**Short answer:** `su` switches your shell to another user (default: root). `sudo` runs one command as another user, with logging and finer-grained policy.
**Deeper:** `su` prompts for the target user's password — so every sysadmin knowing root needs to know root's password, and there's no audit trail. `sudo` asks for your own password, looks up `/etc/sudoers` to see if you're allowed, logs the invocation, and runs just the requested command. In production you almost always use `sudo` for auditability.
**Watch out for:** `sudo su -` is a common anti-pattern that gets you a root shell while bypassing per-command logging.

### Q: How do you find which process is listening on port 8080?
**Short answer:** `ss -tlnp | grep 8080` or `lsof -i :8080`.
**Deeper:** `ss` is the modern replacement for `netstat`. `-t` is TCP, `-l` is listening, `-n` is numeric (no DNS lookups), `-p` shows the owning process. You need root to see other users' processes. If nothing is listening but your request is failing, the problem is upstream — check the firewall (`iptables -L` or `nft list ruleset`) or whether the service even started.
**Watch out for:** Confusing "port is open on the firewall" with "something is listening on the port." Both must be true for a connection to succeed.

### Q: What does `chmod u+s` do and when is it dangerous?
**Short answer:** It sets the **setuid** bit, making the binary run as its *owner* rather than the invoking user. It's classic in `/usr/bin/passwd` (needs root to edit `/etc/shadow`).
**Deeper:** Setuid is how unprivileged users can safely perform privileged actions through a tightly-controlled binary. It's also a massive attack surface — a bug in a setuid-root binary becomes a local privilege escalation. Modern systems prefer Linux capabilities (`setcap`) which grant fine-grained permissions instead of full root.
**Watch out for:** Setuid is ignored on scripts for security reasons (the shebang target runs setuid, not the script). Candidates confidently wrong about this is a common trap.

## Follow-up questions interviewers love

- What happens when a process's parent dies before it does?
- How does the kernel choose which page to swap out under memory pressure?
- Explain the difference between a signal and an interrupt.
- What's in `/proc/<pid>/cmdline` vs `/proc/<pid>/status`?
- Why does `kill -9` sometimes not work? What processes can ignore it?
- How does `tmux` or `screen` keep your session alive after you disconnect?
- What's the practical difference between `/bin/sh` and `/bin/bash`?

## Build with AI — the workflow

1. *"Explain what happens when I run `sudo apt update && sudo apt upgrade`, step by step, including which files change and which processes restart."* — Builds OS intuition.
2. *"I need to find the 10 biggest files on a Linux server. Give me one command, then explain every flag."* — Teaches `find`/`du` idioms without memorisation.
3. *"My service at port 3000 is not reachable from outside the VM. Walk me through a top-down debug plan using only Linux tools."* — Practices the investigative mindset.
4. *"Convert this crontab entry to human-readable English: `*/15 2-5 * * 1-5`."* — Cron is a common interview gotcha.
5. *"Write a 1-minute explanation of Linux capabilities vs setuid, for a senior-engineer audience."* — Lets you rehearse answering a stretch interview question.

## Recap

- Linux is a small set of mental models: files, processes, users, permissions, networks. Learn the models; prompt the commands.
- Fork-exec is how every Linux program starts — know this cold.
- Permissions are a 3×3 bit matrix. `777` is almost always wrong.
- The shell is an interpreter, not magic. `$PATH`, `fork`, `exec`, `wait`.
- Pipes let small tools compose. Senior engineers read one-liners; juniors write them.
- SSH auth is asymmetric cryptography — public key on server, private key never leaves.
- Debugging starts with *where*, not *what*. Tools like `top`, `ss`, `iostat`, `dmesg` narrow the category.

> **Want an AI tutor on this exact lesson plus a full mock DevOps interview?** → **[vibelearner.com/tracks/devops/day/2](https://vibelearner.com/tracks/devops/day/2)**
