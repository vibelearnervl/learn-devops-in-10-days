---
day: 04
title: "Docker Compose & multi-container apps"
duration_min: 60
concepts: ["docker-compose", "networking", "volumes", "multi-container"]
ai_prompts:
  - "I have a Python API, a Postgres database, and a Redis cache. Write a production-grade docker-compose.yml and explain every field."
  - "My web container starts before the database is ready and crashes. Explain healthchecks and how to fix this properly."
  - "When should I stop using Docker Compose and move to Kubernetes? Give me 5 decision criteria."
shorts_ids: []
---

## The one idea for today

**Real applications are never one container.** They're a web app plus a database plus a cache plus a message queue plus a worker. Docker Compose is how you describe that entire system as a single YAML file — one command to start, one command to stop, consistent across every developer's laptop and every CI job.

Compose is not a production orchestrator. It's not meant to compete with Kubernetes. It's meant to be **the fastest way to define and run a multi-container app on one machine**. Every concept you learn here (services, networks, volumes, dependencies) maps directly to Kubernetes tomorrow — Compose is the gentler on-ramp.

## Core concepts

### Services — the unit of composition
A **service** in Compose is a logical component of your app (web, db, cache). Each service is backed by a container (or multiple replicas). You describe the service once: which image to use, which ports to expose, which environment variables to set, which volumes to mount. Compose spawns and manages containers accordingly.

### The compose file is declarative
You don't write `docker run ...` commands. You describe the *desired state* in `docker-compose.yml`. `docker compose up` figures out what to create, start, or restart to reach that state. `docker compose down` tears it all back to zero. This declarative model is exactly how Kubernetes works at a larger scale.

### Service discovery — the killer feature
When two services are on the same Compose network, they can reach each other by **service name**. Your Django container connects to `postgres:5432` — no IPs, no DNS configuration. Compose creates a user-defined bridge network and runs an embedded DNS server that resolves service names to the container's current IP. This is service discovery in miniature.

### Networks — logical, not physical
By default, Compose creates one network per project and attaches every service to it. You can define multiple networks to isolate services (e.g. a `frontend` net for web + proxy, a `backend` net for web + db, preventing the proxy from ever seeing the database). Networks are just Linux bridges under the hood.

### Volumes — persistence across `down` and `up`
When a container is destroyed, its writable filesystem is gone. Volumes survive. Define named volumes at the top level (`volumes: { pgdata: {} }`), mount them into services (`volumes: ["pgdata:/var/lib/postgresql/data"]`), and your Postgres data outlives restarts. Never store production data only inside the container's writable layer.

### Environment variables and `.env` files
Compose expands `${VAR}` from the shell or from a `.env` file in the same directory. The pattern: keep secrets out of `docker-compose.yml`, put them in `.env`, `.gitignore` that file. Services receive env via `environment:` (explicit) or `env_file:` (load many from a file).

### `depends_on` vs healthchecks — the race condition
`depends_on: [db]` only guarantees **start order**, not readiness. Your web container can start "after" Postgres but still fail because Postgres isn't accepting connections yet. The fix: add a `healthcheck:` block to Postgres (`pg_isready`), then have web `depends_on:` with a `condition: service_healthy` clause. This is the single most common Compose bug.

### Scaling a service
`docker compose up --scale web=3` runs three replicas of `web`. Compose load-balances via DNS round-robin. This is how you prototype horizontal scaling on one machine. For real load balancing, put a reverse proxy (Traefik, Caddy, nginx) in front.

### Override files
`docker-compose.yml` is your base. `docker-compose.override.yml` is merged on top automatically — perfect for local-only tweaks (mount source code, enable debug flags). `docker-compose.prod.yml` is loaded explicitly with `-f`. This split lets one codebase run in dev and prod without duplication.

### Dev vs prod with Compose
In dev: bind-mount source, expose debugger ports, use `command:` overrides for hot reload. In prod: no bind mounts (your image must be self-contained), set restart policies, tune healthchecks, disable published ports except the entry point. Same file, different overrides.

### When to graduate from Compose to Kubernetes
Compose breaks down at: multiple hosts, zero-downtime rolling updates, auto-scaling on metrics, declarative secrets management, multi-team isolation (namespaces), certificate automation at scale. If you need any two of those, you're ready for Kubernetes.

## Common gotchas

- Forgetting `depends_on: condition: service_healthy` — everything "starts" but fails intermittently
- Hard-coding container IPs in config — always use service names
- Exposing database ports to the host (`ports: ["5432:5432"]`) in prod — classic attack vector
- Defining volumes inline (`./data:/data`) and then wondering why data lives in your source tree
- Mixing Compose file versions (v2 vs v3 vs no-version) — stick to the modern no-version schema
- Using `restart: always` everywhere — it hides bugs that a crash loop would expose
- Running `docker compose up` without `-d` in a service context and wondering why it dies when you close the terminal

## Interview questions with model answers

### Q: What is Docker Compose and when is it the right tool?
**Short answer:** Compose is a tool for defining and running multi-container apps on a single host, via a declarative YAML file.
**Deeper:** It's ideal for three jobs: local development of multi-service apps, CI pipelines that need an ephemeral stack (run tests against a real Postgres, tear it down), and simple single-host production deployments. Where it breaks down is multi-host scheduling, rolling deploys, auto-scaling, and secret management — those are Kubernetes territory. A useful mental model: Compose gives you 80% of Kubernetes's *vocabulary* with 1% of its operational overhead, for workloads that fit on one machine.
**Watch out for:** Candidates who say Compose is "production-ready" universally — it is for a single-host hobby deploy, it isn't for multi-host critical systems.

### Q: How do containers in a Compose project discover each other?
**Short answer:** Compose creates a user-defined bridge network and runs an embedded DNS server. Containers resolve each other by service name — `db` resolves to whichever IP that service is currently at.
**Deeper:** When you define `services: web, db`, Compose puts both containers on the same internal network. Each service's name becomes a DNS entry. When `web` calls `http://api:8000` or `db:5432`, Docker's DNS resolver (listening at 127.0.0.11 inside each container) returns the container's current IP. Because names are stable and IPs aren't, you never hard-code IPs. Scale a service to multiple replicas and the DNS record returns multiple IPs — clients get load balancing for free via round-robin.
**Watch out for:** Saying "they share a network" is technically right but shallow; the DNS layer is what makes names work.

### Q: How do you handle a service that depends on a database being ready, not just started?
**Short answer:** Use `healthcheck` on the database plus `depends_on: condition: service_healthy` on the dependent service.
**Deeper:** `depends_on` alone only orders starts — Compose will start the db container first, then web — but the db process takes seconds to open its listening socket. Your web app crashes on connect. With `healthcheck: [test: ["CMD-SHELL", "pg_isready -U postgres"], interval: 5s, retries: 5]`, Compose probes the db until it answers. Then it marks the service `healthy`. The dependent service's `depends_on: db: { condition: service_healthy }` waits for that healthy state before starting. For apps where you can't modify the upstream image, use `wait-for-it.sh` or `dockerize` as the entrypoint wrapper.
**Watch out for:** Retry-in-app-code is also acceptable — some interviewers prefer that because it handles mid-flight database restarts, not just startup.

### Q: Compare bind mounts and named volumes in a Compose file.
**Short answer:** Bind mounts map a host path into the container; volumes are named locations managed by Docker. Bind mounts for dev code-sync; volumes for production persistence.
**Deeper:** `volumes: ["./app:/code"]` is a bind mount — great for hot reload during development. `volumes: ["pgdata:/var/lib/postgresql/data"]` (with `pgdata` defined at the top level) is a named volume — Docker controls the storage location, handles permissions, and can be backed by drivers (NFS, cloud volumes). Bind mounts couple your stack to a specific host filesystem layout, which breaks portability; volumes travel with the Compose project. In production, you never bind-mount your code (code lives in the image), and you always volume-mount your data.
**Watch out for:** Candidates say "volumes are faster" — not necessarily true on Linux; it's the *management* difference that matters.

### Q: How do you prevent secrets from ending up in your Compose file?
**Short answer:** Put them in a `.env` file next to `docker-compose.yml`, reference with `${VAR}` substitution, and `.gitignore` the `.env`.
**Deeper:** For richer production setups, Compose supports `secrets:` blocks (Swarm mode / Compose v3.1+), which mount secrets as tmpfs files inside containers rather than environment variables (which leak into `docker inspect`, process listings, and logs). The order of preference: CI-provided env > Docker secrets > `.env` file > hardcoded. Never commit a `.env` file containing real credentials; commit a `.env.example` with keys but placeholder values instead.
**Watch out for:** Env-var secrets are convenient but globally visible to the container process — anyone with `docker exec` or log access can read them.

### Q: How do you run the same Compose project in dev and prod without duplicating YAML?
**Short answer:** Use a base `docker-compose.yml` plus override files — `docker-compose.override.yml` (auto-loaded for dev) and `docker-compose.prod.yml` (loaded explicitly for prod).
**Deeper:** Compose deep-merges override files into the base. The base defines stable fields (image, depends_on, healthchecks). The dev override adds bind mounts, debug ports, hot-reload commands. The prod override hardens config (restart policies, removed ports, stricter healthchecks). You launch prod with `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d`. This pattern avoids two drifting copies of the stack.
**Watch out for:** Candidates forget that Compose auto-loads `docker-compose.override.yml` when present — so the "dev config" can accidentally apply in CI if you don't exclude it.

### Q: Your multi-service app starts fine locally but intermittently fails in CI. Where do you look?
**Short answer:** Most likely a startup race — a dependent service connects before its dependency is actually accepting connections.
**Deeper:** CI runners are faster or slower than laptops, exposing race conditions. Check: do you have healthchecks on data services? Are dependents using `condition: service_healthy`? Is your app code retrying on connect failure? Next, check resource limits — CI runners often have less RAM and slower disks, so a Postgres that starts in 2s locally might take 10s in CI. Also check that `.env` variables required in prod aren't missing in CI — Compose doesn't fail loudly when a `${VAR}` is empty.
**Watch out for:** Candidates jump to "flaky test" without inspecting the startup sequence.

### Q: When should you NOT use Docker Compose?
**Short answer:** When you need multi-host scheduling, zero-downtime rolling updates, auto-scaling on metrics, first-class secret management, or team-level isolation.
**Deeper:** Compose is a single-host tool. If your app needs to survive a node failure, you need an orchestrator. If your deploy needs to cut over without downtime (blue-green, canary), Compose can only stop and start. If you need to enforce resource quotas per team or namespace, you need Kubernetes or a managed platform. A cleaner heuristic: Compose is perfect for dev, CI, and simple single-host hobby deployments. Anything with an SLA or paying users typically deserves Kubernetes or a PaaS.
**Watch out for:** The temptation to "grow" Compose with bash scripts wrapping `compose up` on multiple hosts — that's always a worse k8s.

## Follow-up questions interviewers love

- How does Docker's embedded DNS server work?
- What's the difference between `expose:` and `ports:` in a Compose file?
- How do you implement zero-downtime deploys using only Compose?
- Why might two containers on the same Compose network still fail to reach each other?
- What happens to your data if you `docker compose down -v`?
- How do you run a one-off task (like a DB migration) in a Compose setup?
- How is Compose v2 (Go-based, `docker compose`) different from Compose v1 (Python-based, `docker-compose`)?

## Build with AI — the workflow

1. *"Write a production-grade `docker-compose.yml` for a Django app with Postgres and Redis. Include healthchecks, a non-root user in the web service, and a named volume for the database. Explain every line."* — Builds a real artefact.
2. *"My containers start but the web service can't connect to the database on the first try. Walk me through a diagnostic plan."* — Reinforces the race-condition pattern.
3. *"Explain how Compose's service discovery works, from the DNS query inside the container to the IP resolution. Pitch it like I'm preparing to answer a staff-engineer question."* — Depth practice.
4. *"Convert this Compose file to an equivalent Kubernetes manifest set. Explain which parts map 1:1 and which don't."* — Bridges today to tomorrow.
5. *"Audit this Compose file for security issues."* (paste yours) — Teaches code review, not just writing.

## Recap

- Compose = declarative YAML for multi-container apps on one host.
- Services discover each other by name via Compose's DNS.
- `depends_on` starts in order; use healthchecks for *readiness*.
- Volumes persist data across `down` and `up`; bind mounts map host paths.
- Override files are how one Compose project supports dev, CI, and prod.
- Graduate to Kubernetes when you need multi-host, rolling updates, or auto-scaling.

> **Want an AI tutor on this exact lesson plus a full mock DevOps interview?** → **[vibelearner.com/tracks/devops/day/4](https://vibelearner.com/tracks/devops/day/4)**
