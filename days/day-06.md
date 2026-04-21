---
day: 06
title: "CI/CD pipelines — GitHub Actions, concepts over YAML"
duration_min: 60
concepts: ["ci-cd", "github-actions", "pipelines", "automation"]
ai_prompts:
  - "Design a CI/CD pipeline for a Node.js app with Docker builds, tests, and staging + production deploys. Explain every stage and why it exists."
  - "My CI pipeline takes 25 minutes. Give me a diagnostic plan and a ranked list of optimisations that typically work."
  - "Compare CI, CD (delivery), and CD (deployment). When is each the right target for a team?"
shorts_ids: []
---

## The one idea for today

**CI/CD is not a YAML skill — it's a feedback-loop skill.** Continuous Integration means: every commit gets built and tested automatically, so a break is caught in minutes, not at release time. Continuous Delivery means: every green build produces a shippable artefact. Continuous Deployment means: every green build automatically goes to production.

Every CI platform — GitHub Actions, GitLab CI, CircleCI, Jenkins — uses the same grammar: **triggers → jobs → steps, running on runners, with secrets and cache**. Once you understand the grammar, switching platforms is a weekend, not a rewrite. Memorising YAML syntax is not the skill; designing pipelines that are fast, safe, and trustworthy is.

## Core concepts

### CI vs CD (delivery) vs CD (deployment)
**CI** = build and test on every push. **Continuous Delivery** = every green build produces an artefact ready to deploy (you click a button). **Continuous Deployment** = every green build goes to production automatically. Teams typically earn the right to progress through these stages — you don't start at deployment.

### The universal pipeline grammar
- **Trigger** — push, pull request, tag, schedule, manual dispatch
- **Job** — a set of steps that runs on one runner
- **Step** — a single command or action
- **Runner** — the VM or container that executes the job
- **Artefact** — a file passed between jobs or out of the pipeline
- **Cache** — persistent data (dependencies) reused across runs
- **Secret** — encrypted value injected at runtime

Every CI platform is a different spelling of this grammar.

### Ephemeral runners
Each job starts in a fresh environment. This is a *feature*, not a bug — it guarantees that "works on my machine" can't happen. The tradeoff: cold starts. That's why caching is so important.

### Matrix builds — fan out across variants
A matrix runs the same job across combinations of variables: Node 18 + 20 + 22 × Linux + macOS. You write the job once and get 6 parallel runs. Great for libraries that must work across environments; overkill for a single-target app.

### Caching is the single biggest speed win
A fresh `npm ci` might take 2 minutes. Cached, 10 seconds. Same for `pip`, Maven, Go modules, Docker layers. GitHub Actions uses `actions/cache` with a key (usually a hash of your lockfile). A cache miss is expensive; a cache hit is free. Knowing how to design cache keys is a real skill.

### Artefacts and job handoff
The build job produces a JAR; the deploy job needs it. You upload it as an artefact in the build job, download it in the deploy job. For container images, the "artefact" is the image pushed to a registry — the deploy job pulls it back.

### Environments and approvals
Production deploys shouldn't happen silently. GitHub's "environments" feature lets you require a manual approval before a job runs, restrict deploys to specific branches, and inject environment-specific secrets. This is how you get CD without losing a safety net.

### Secrets and OIDC — the supply-chain shift
Old way: paste a long-lived AWS key as a repo secret. Risk: if the repo is compromised, the key is exfiltrated. New way: **OIDC federation** — GitHub mints a short-lived token per run, AWS trusts GitHub's identity provider, no long-lived credentials leave the vault. This is the current best practice for cloud deploys.

### Pinning actions for supply-chain safety
A third-party action like `actions/checkout@v4` is a moving target. For sensitive pipelines, pin to a **commit SHA** — `uses: actions/checkout@8f4b7f8...` — so a malicious new tag can't silently change behaviour.

### Path filters and affected-only builds
In a monorepo, you don't want every pipeline running on every commit. `on.push.paths: ["services/api/**"]` limits triggers to relevant changes. For Turborepo-style "affected" detection, tools like Nx compute the dependency graph and run only downstream jobs.

### Reusable workflows and composite actions
Don't copy-paste YAML across repos. GitHub's **reusable workflows** (`workflow_call`) let one workflow invoke another. Composite actions package sequences of steps as a single reusable unit. This is how platform teams offer "golden" CI paths that product teams can adopt.

### The three pipeline failure modes
A pipeline fails because **(1)** the code genuinely broke (good), **(2)** the pipeline infrastructure had a hiccup — flake, network blip (bad, noisy), or **(3)** the pipeline is poorly designed — race conditions, implicit ordering (fix root cause). Telling these apart is a senior skill.

## Common gotchas

- Running tests on `push` *and* `pull_request` — doubles your CI cost with no extra safety
- Using `latest` for action versions — supply-chain risk
- Storing secrets as plain env vars in the YAML — they end up in logs
- Building the Docker image in every job instead of once, then promoting the tag
- Forgetting to run `npm ci` (not `npm install`) in CI — lockfile mismatch = non-reproducible builds
- "Re-run failed jobs" as a coping mechanism for flakes instead of fixing them
- No concurrency controls — two merges to main kick off two production deploys racing each other
- Long-running jobs without a timeout — one stuck job blocks the whole pipeline

## Interview questions with model answers

### Q: What's the difference between CI, continuous delivery, and continuous deployment?
**Short answer:** CI = tested on every commit. Continuous Delivery = every green build is ready to ship with a manual gate. Continuous Deployment = every green build goes to prod automatically.
**Deeper:** The progression reflects confidence. CI requires good tests. Continuous Delivery additionally requires a reliable build pipeline that produces deployable artefacts and clean rollback. Continuous Deployment additionally requires excellent observability and automated rollback — because you can't manually oversee every deploy, the system has to catch regressions itself. Most teams never reach deployment; that's fine. What matters is the team's throughput and lead time metrics, not which word they use.
**Watch out for:** Candidates who conflate Delivery and Deployment. The interview signal is whether they can articulate the *gate* between them.

### Q: Design a CI/CD pipeline for a Node.js microservice. What stages would you have?
**Short answer:** Lint + type check → unit tests → build Docker image → container security scan → integration tests against the image → push to registry → deploy to staging → smoke tests → manual approval → deploy to prod.
**Deeper:** The stages must be ordered by cost and feedback value — fast, cheap checks first (lint, unit tests in under 2 minutes), expensive checks later (integration tests, security scans). The pipeline should fail fast: a syntax error shouldn't wait for a full Docker build. Parallelise independent stages (lint and type-check can run simultaneously). Build once, promote the same immutable image through every environment — never rebuild per environment. Smoke tests in staging catch environment-specific regressions that unit tests miss. The manual approval in front of prod is a deliberate friction point — teams earn the right to automate it away with better tooling.
**Watch out for:** Candidates who skip the "build once, promote the artefact" rule — rebuilding per environment invites environment drift.

### Q: Why is pipeline caching so important, and how do you design good cache keys?
**Short answer:** Fresh installs dominate pipeline time; a cache hit saves minutes. A good key is deterministic and invalidates when inputs change.
**Deeper:** A good key looks like `deps-${{ runner.os }}-node20-${{ hashFiles('**/package-lock.json') }}`. Runner OS in the key because artefacts built on Linux aren't portable to macOS. Language version because a `node_modules` built for Node 18 misbehaves on Node 20. Lockfile hash because that's the exact dependency set. You also define a **restore-keys** chain — falls back to a partial match if no exact key exists, so CI still benefits from a stale-but-close cache on the first run after an upgrade. Poor keys cause either *too many hits* (incorrect cached artefacts, silent bugs) or *too few* (pipeline is slow).
**Watch out for:** Using `hashFiles('**/*.js')` as a key — source changes invalidate the dependency cache, defeating its purpose.

### Q: How do you handle secrets in a CI pipeline safely?
**Short answer:** Never in YAML. Use the platform's encrypted secret store. For cloud credentials, prefer OIDC federation over long-lived keys.
**Deeper:** Plain env vars in the workflow file leak into logs, PRs, and forks. GitHub Actions masks secrets in logs but a `echo $SECRET | base64` defeats the mask. The right layers: (1) use OIDC to avoid any long-lived credential where possible — GitHub mints a short-lived token per run that AWS/GCP/Azure trusts. (2) For inevitable long-lived secrets (third-party API keys), store them as encrypted repo or environment secrets. (3) Scope to environment: prod secrets only available on prod-deploy jobs, gated by branch and reviewer. (4) Rotate them on a schedule and after any contractor offboarding. (5) For really high-value secrets, reach for a dedicated manager (Vault, AWS Secrets Manager) with short-lived leases.
**Watch out for:** "We just use repo secrets" is fine for a small team but shouldn't be the only answer in a senior interview.

### Q: Your pipeline is flaky — fails randomly. How do you diagnose and fix it?
**Short answer:** Measure flake rate first, then categorise: environment, concurrency, or test code. Fix root causes, don't add retries.
**Deeper:** Start by adding a flake dashboard — which jobs and which tests fail intermittently. Categories: **environmental** (network blip, runner disk full, registry 5xx) — add narrow retries only on those specific steps; **concurrency** (two jobs writing the same file, port collisions) — isolate per-job workspaces, use randomised ports; **test code** (time dependencies, fixture leakage, async races) — fix the tests. The anti-pattern is blanket `retry: 3` — it hides all three categories and lets bad tests accumulate. Senior engineers push back on "just re-run" as a culture.
**Watch out for:** Candidates who add retries as the only fix — a yellow flag.

### Q: What is OIDC federation in CI and why is it better than long-lived cloud keys?
**Short answer:** OIDC lets your CI runner prove its identity to a cloud provider (AWS/GCP/Azure) without carrying a static key. The cloud trusts GitHub's identity provider and issues a short-lived token.
**Deeper:** Your GitHub workflow requests a token from GitHub's OIDC provider. That token includes verified claims (repo, branch, workflow, actor). You configure the cloud IAM role's trust policy to accept tokens only with specific claims — e.g. "only the `deploy-prod` workflow in the `myorg/myrepo` repo on the `main` branch can assume me." GitHub exchanges its OIDC token for cloud credentials via STS. Those creds are valid for minutes, not months. If the repo is compromised, an attacker can't exfiltrate a stale AWS key because there are no stale keys — the next token request is gated by the claim check.
**Watch out for:** Candidates who've heard of OIDC but can't explain the claim-based trust model.

### Q: How do you prevent two merges to main from racing each other to production?
**Short answer:** Concurrency groups. GitHub Actions has `concurrency: { group: deploy-prod, cancel-in-progress: false }` which queues or cancels overlapping runs.
**Deeper:** Without concurrency control, commit A and commit B merge within seconds, two deploy jobs run in parallel, and one overwrites the other's infrastructure changes or container image tag — effectively non-deterministic production state. `cancel-in-progress: true` is right for build/test (cancel the earlier run when a newer commit lands). `cancel-in-progress: false` is right for deploys (queue them so every commit actually deploys in order). Beyond that, modern deploys use immutable, digest-pinned artefacts so even a race produces a deterministic end state — the last commit wins, but nothing partial survives.
**Watch out for:** "We deploy one at a time" as a hand-waved answer without knowing the mechanism.

## Follow-up questions interviewers love

- How do you bootstrap a new repo's CI in under 30 minutes?
- What's the difference between a GitHub Actions composite action and a reusable workflow?
- How would you implement a canary deployment using only CI primitives?
- What is Dependabot and what problem does it solve?
- How do self-hosted runners change the security model?
- Why would you want to run CI against the post-merge state (on main) even if PR builds passed?
- How do you prevent a malicious PR from an outside contributor from exfiltrating your secrets?

## Build with AI — the workflow

1. *"Write a GitHub Actions workflow for a Python FastAPI app that runs lint, unit tests, builds a Docker image, scans it with Trivy, and deploys to staging. Include caching and OIDC auth to AWS. Explain every choice."* — Production artefact practice.
2. *"My test job is slow. Here are the logs and the workflow YAML. Give me a ranked list of optimisations with estimated impact."* — Teaches diagnostic thinking.
3. *"Explain OIDC federation between GitHub Actions and AWS, as if I'm answering a principal-engineer interview question."* — Depth rehearsal.
4. *"Here's our monorepo of 12 services. Design the CI so that changing one service doesn't rebuild all 12."* — Monorepo architecture practice.
5. *"Audit this GitHub Actions workflow for supply-chain and secret-leakage risks."* — Security-review muscle.

## Recap

- CI = automated tests on every push. CD (delivery) = ready to ship. CD (deployment) = auto to prod.
- Every platform uses the same grammar: triggers, jobs, steps, runners, artefacts, caches, secrets.
- Cache design is the #1 speed lever. Bad keys are worse than no cache.
- Build once, promote the artefact. Never rebuild per environment.
- OIDC > long-lived keys for cloud deploys.
- Pin actions by SHA in sensitive pipelines. `@latest` is a supply-chain risk.
- Concurrency groups prevent parallel prod deploys. Fix flakes, don't retry them.

> **Want an AI tutor on this exact lesson plus a full mock DevOps interview?** → **[vibelearner.com/tracks/devops/day/6](https://vibelearner.com/tracks/devops/day/6)**
