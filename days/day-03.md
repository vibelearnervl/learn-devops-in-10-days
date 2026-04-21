---
day: 03
title: "Docker — containers, images, and layers explained"
duration_min: 60
concepts: ["docker", "containers", "images", "layers"]
ai_prompts:
  - "Explain the difference between a container and a virtual machine using an analogy that even my non-technical friend would understand."
  - "Walk me through what happens inside the Linux kernel when `docker run nginx` executes — namespaces, cgroups, overlay filesystem."
  - "Optimise this Dockerfile for a Node.js app. Tell me every change you make and why, ordered by impact."
shorts_ids: []
---

## The one idea for today

**A container is not a tiny virtual machine. It's a regular Linux process that's been lied to about what it can see.** Docker uses kernel features — namespaces to limit what the process sees (its own PIDs, filesystem, network), cgroups to limit what it can consume (CPU, memory, I/O) — to make one process feel like it owns a whole OS. There is no guest kernel. There is no virtualisation. It's just Linux being clever.

Once this clicks, every weird Docker behaviour stops being weird. Why is the image so small? Because it's just a tarball of files. Why does my container not see the host's processes? Because the PID namespace hides them. Why does networking feel magical? Because it's virtual interfaces wired to the host's bridge.

## Core concepts

### Container = isolated process
When you run `docker run nginx`, Docker starts an nginx process on your host, wrapped in namespaces that hide everything else. Inside the container, `ps` shows only the container's processes. The network stack looks private. The filesystem is mounted from the image. But it's all one Linux kernel underneath.

### Image = layered read-only filesystem
A Docker image is not an ISO. It's a **stack of tarballs**, each layer representing changes on top of the previous layer. When the container starts, Docker mounts these layers as a read-only union filesystem, then adds a writable layer on top. This is why an Ubuntu image can be 70 MB and a Python image built from it is only another 50 MB — you're just adding layers.

### Dockerfile = build instructions, one layer per step
Each instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, `CMD`) creates one layer. Layers are cached. If you change line 15, layers 1–14 are reused; layers 15+ rebuild. This is why instruction order matters — put things that change rarely (dependencies) before things that change often (your source code).

### Container lifecycle
`docker create` → stopped state. `docker start` → running. `docker stop` → sends SIGTERM, waits, then SIGKILL. `docker rm` → removes the stopped container. `docker ps` shows running; `docker ps -a` shows all. Understanding this state machine prevents "I stopped it but it's still showing up" confusion.

### Registries and tags
A **registry** hosts images. Docker Hub is the default. An image reference like `nginx:1.25-alpine` is `<registry>/<repo>:<tag>` — `docker.io/library/nginx` is implied. Tags are mutable labels; the immutable identifier is the **digest** (`sha256:...`). Production deploys should pin by digest, not tag.

### Bind mounts vs volumes
A **bind mount** maps a host directory into the container — great for dev ("edit on host, see changes in container"). A **volume** is a Docker-managed location — better for production data because Docker owns the lifecycle, permissions, and portability. Bind mounts bleed host paths into your image; volumes don't.

### Container networking — bridge, host, none
The default `bridge` network creates a virtual switch on the host. Each container gets a virtual NIC plugged into that switch. `host` mode shares the host's network namespace (no isolation). `none` gives the container no network at all. Behind the scenes it's just `iptables` rules and `veth` pairs.

### Multi-stage builds
A Dockerfile can have multiple `FROM` statements. Each one starts a new stage. The `COPY --from=0` syntax lets a later stage pull files from an earlier one. This is how you build a Go binary in a 1 GB "builder" image and copy it into a 10 MB "runner" image — the builder never ships.

### Image size is a security and cost concern
Bigger images mean slower pulls, more disk, more attack surface. The fastest wins: use `alpine` or `distroless` bases, delete apt caches in the same `RUN` layer, use multi-stage builds, `.dockerignore` your build context.

### Exit codes and PID 1
The **entrypoint process inside the container is PID 1** — which means it's responsible for reaping zombie children and responding to signals. Many apps don't know this (they assume the OS handles it), causing graceful shutdown bugs. `tini` or `docker run --init` injects a proper init.

### Docker vs container runtime
Docker is the user-friendly CLI + daemon. Under the hood, it delegates to `containerd`, which delegates to `runc`. Kubernetes talks directly to `containerd` (or CRI-O), skipping Docker entirely. "Docker" is a brand; "OCI container" is the standard.

## Common gotchas

- Layer order matters: put `COPY package.json` before `COPY . .` so `npm install` is cached when only source changes
- `.dockerignore` is mandatory — without it, your build context can be gigabytes of `node_modules`
- Every `RUN` makes a layer. Chain commands with `&&` and clean caches (`apt clean`) in the same `RUN`
- Running as root inside a container is the default and is a security smell. Use `USER` directive
- `CMD` vs `ENTRYPOINT` confusion: `ENTRYPOINT` is the binary; `CMD` is default args. `docker run` args override `CMD` but not `ENTRYPOINT`
- Environment variables set at build time (`ENV`) leak into the image — never bake secrets into Dockerfiles
- `latest` tag is a lie. It doesn't mean "newest" — it means "whatever the publisher most recently tagged as latest"

## Interview questions with model answers

### Q: What is the real difference between a container and a VM?
**Short answer:** A VM virtualises *hardware* — it runs its own kernel on top of a hypervisor. A container virtualises the *OS* — it shares the host kernel but uses namespaces and cgroups to create isolation.
**Deeper:** Because a VM has its own kernel, it needs more RAM (hundreds of MB just to boot), starts slowly (tens of seconds), and isolates more strongly. A container shares the host kernel, so it starts in milliseconds and uses tens of MB, but a kernel vulnerability can escape containers more easily than VMs. The practical consequence: you'd run untrusted code from a random customer in a VM (or a Firecracker microVM), not a plain Docker container. You'd run your own microservices in containers because speed and density matter more than hypervisor-grade isolation.
**Watch out for:** Candidates say "containers don't have an OS" — wrong. They have a user space (libc, bash, etc.); they just share the host's kernel.

### Q: What happens when I run `docker run nginx`?
**Short answer:** Docker pulls the `nginx` image if missing, creates a new layered filesystem on top of it, creates Linux namespaces for isolation, sets up networking, and launches the image's entrypoint as PID 1 inside those namespaces.
**Deeper:** The Docker daemon checks if `nginx:latest` is in its local image cache. If not, it talks to the registry, downloads each layer in parallel (verifying sha256 digests), and extracts them. It then creates a new thin writable layer on top using overlayfs. It calls `clone()` with flags to create new PID, network, mount, UTS, and IPC namespaces. It sets up a virtual ethernet pair — one end in the container's network namespace, the other plugged into the `docker0` bridge. Finally it execs nginx in the new namespaces. To the outside world, it's a normal Linux process. To nginx, it looks like it owns the machine.
**Watch out for:** "Docker creates a small VM" is wrong. There's no hypervisor, no guest kernel.

### Q: Explain Docker layer caching. Why does the Dockerfile order matter?
**Short answer:** Each instruction produces a layer that Docker caches by content hash. If the inputs to a layer are unchanged, Docker reuses the cached layer. Changing an earlier layer invalidates all subsequent layers.
**Deeper:** Consider a Node app. If you `COPY . .` before `RUN npm install`, every code change invalidates the `npm install` layer, forcing a full reinstall. If you `COPY package*.json ./` first, then `RUN npm install`, then `COPY . .`, the install layer only rebuilds when dependencies change. The pattern generalises: copy dependency manifests, install, then copy source. This single change often drops build times from minutes to seconds in CI.
**Watch out for:** Forgetting that `ARG` and build-time variables also participate in cache keys — changing an `ARG` busts everything below it.

### Q: What's a multi-stage build and when should you use it?
**Short answer:** A Dockerfile with multiple `FROM` statements, where later stages copy artefacts from earlier stages. You use it to keep build tools out of the final image.
**Deeper:** Classic case: Go or Rust. Stage 1 uses a full SDK image (1+ GB), compiles the binary. Stage 2 starts from `scratch` or `distroless/static`, copies just the binary. Final image is a few megabytes, with zero attack surface beyond your binary. The builder stage never ships. Same pattern for Node (build with full toolchain, run with `node:alpine`), Python (install wheels, run with slim), Java (build with Maven, run with a JRE).
**Watch out for:** Using the same base image for both stages defeats the point — you need a *smaller* runtime image.

### Q: Why shouldn't containers run as root?
**Short answer:** If the container is compromised or escapes, the attacker has root on the host (or close to it). Running as a non-root user contains the blast radius.
**Deeper:** By default, UID 0 in the container maps to UID 0 on the host unless user namespaces are enabled. A kernel bug or a misconfigured bind mount can turn container-root into host-root. Beyond escape, running as root means any malware or supply-chain attack has write access to every file in the image, can install packages, can bind low-numbered ports, can bypass RBAC checks in orchestrators. `USER 1000` in your Dockerfile, setting ownership on your app files, and running behind a reverse proxy for privileged ports — these are the standard mitigations.
**Watch out for:** Some base images silently run as root; check with `docker inspect` before trusting.

### Q: What's the difference between `CMD` and `ENTRYPOINT`?
**Short answer:** `ENTRYPOINT` is the command that always runs. `CMD` supplies default arguments that the user can override on `docker run`.
**Deeper:** `ENTRYPOINT ["nginx"]` + `CMD ["-g", "daemon off;"]` means: by default the container runs `nginx -g "daemon off;"`. If someone does `docker run image -v`, it becomes `nginx -v` — args override `CMD` but `ENTRYPOINT` stays. Use `ENTRYPOINT` when the container has one clear purpose (a specific service). Use `CMD` alone when the image is a general-purpose toolbox that users might invoke differently. Mixing shell form (`CMD foo bar`) and exec form (`CMD ["foo","bar"]`) breaks signal handling — always prefer exec form.
**Watch out for:** Shell form runs through `/bin/sh -c`, meaning your real process is PID 2, not PID 1 — signals don't reach it properly.

### Q: How do Docker volumes differ from bind mounts?
**Short answer:** Volumes are managed by Docker (stored in `/var/lib/docker/volumes/`); bind mounts map an arbitrary host directory into the container. Volumes are portable; bind mounts are tied to the host filesystem.
**Deeper:** For production data (databases, uploads), volumes are the right choice — Docker handles cleanup, permissions, and backup hooks, and the data isn't coupled to a specific host path. For local dev, bind mounts win because you want to edit code on the host and see changes instantly inside the container. Volumes support drivers (NFS, cloud storage), so you can plug in networked storage without changing your app. Bind mounts assume the host layout, which hurts portability across dev/stage/prod.
**Watch out for:** On Docker Desktop (Mac/Windows), bind mounts are slow because they cross the VM boundary — use volumes or Mutagen-backed mounts for performance.

### Q: What is PID 1 in a container, and why does it matter?
**Short answer:** The entrypoint process inside the container becomes PID 1 in its namespace. PID 1 has special responsibilities — reaping zombies and handling signals — that most application binaries aren't written to perform.
**Deeper:** In a normal Linux system, PID 1 is `init` (systemd, sysvinit). It reaps orphaned processes and handles SIGTERM gracefully. When you run `node server.js` as PID 1 in a container, Node wasn't designed to reap zombies, so they pile up. Also, Node doesn't forward SIGTERM to child processes properly, so `docker stop` waits the full 10s and then SIGKILLs everything. The fix is `tini` (or `docker run --init`), a 100 KB init that sits at PID 1 and exec's your real process as PID 2, handling signals and reaping correctly.
**Watch out for:** Candidates who've never seen this bug in production won't mention it — senior interviewers use it as a filter.

## Follow-up questions interviewers love

- What's the difference between `docker run -d` and `docker run`?
- How does `docker exec` work under the hood?
- What is overlayfs and why does Docker use it?
- Why is a `scratch` base image useful, and what are its limits?
- How would you debug a container that exits immediately on startup?
- What's the right way to pass secrets into a container at runtime?
- How do you implement log rotation for containers?
- What happens to a container when the Docker daemon restarts?

## Build with AI — the workflow

1. *"Give me a Dockerfile for a Python Flask app, using multi-stage build and a non-root user. Explain every line."* — Practices production patterns.
2. *"My Node container is 1.2 GB. Here's the Dockerfile [paste]. Rewrite it to be under 100 MB and list every optimisation."* — Teaches image size thinking.
3. *"Walk me through the Linux namespaces Docker uses. For each, give one concrete thing it isolates and how to opt out of that isolation."* — Builds kernel intuition.
4. *"I ran `docker run myapp` and nothing seems to be happening. List the 10 most likely causes in decreasing order of probability, with the command to diagnose each."* — Practices debugging mindset.
5. *"Explain Docker's overlayfs storage driver like I'm preparing for a principal-engineer interview."* — Stretch-target explanation practice.

## Recap

- Container = isolated process, sharing the host kernel. Not a tiny VM.
- Images are stacks of layers. Dockerfile order controls cache efficiency.
- Multi-stage builds keep compilers out of production images.
- Bind mounts for dev, volumes for prod data.
- Never run containers as root in production; fix PID 1 with `--init`.
- `CMD` is overridable default args; `ENTRYPOINT` is the fixed command.
- `latest` is a trap; pin by digest for real reproducibility.

> **Want an AI tutor on this exact lesson plus a full mock DevOps interview?** → **[vibelearner.com/tracks/devops/day/3](https://vibelearner.com/tracks/devops/day/3)**
