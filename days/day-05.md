---
day: 05
title: "Kubernetes concepts — pods, services, deployments"
duration_min: 75
concepts: ["kubernetes", "pods", "services", "deployments"]
ai_prompts:
  - "Explain what happens inside Kubernetes when I run `kubectl apply -f deployment.yaml` — every component that gets involved, in order."
  - "Compare Pod, ReplicaSet, Deployment, and StatefulSet using a restaurant franchise analogy. Make it stick."
  - "I need to expose my web app to the public internet on Kubernetes. Walk me through every object I need, from Pod to public URL."
shorts_ids: []
---

## The one idea for today

**Kubernetes is a control loop that makes reality match your description.** You hand it YAML that says "I want 3 replicas of my app, exposed on port 80, with this health check." Kubernetes checks: does reality match? If not, it creates pods, restarts failed ones, replaces dead nodes, reroutes traffic — forever, without you. You're not running containers anymore; you're running a system that runs containers.

The reason Kubernetes feels complicated is that it ships with *many* kinds of objects (Pod, Service, Deployment, ConfigMap, Ingress, PV, PVC…). But the pattern is always the same: you declare desired state, a controller reconciles reality toward it. Master that loop and everything else clicks.

## Core concepts

### The control loop — the soul of Kubernetes
Every Kubernetes controller runs a loop: *observe current state → compare to desired state → take action to close the gap → repeat*. This is why you `apply` manifests, not `create` them. You describe the end; Kubernetes does the driving. It's the same model as a thermostat: you set the target, the system keeps adjusting.

### Nodes and the control plane
A cluster is made of **nodes** (machines — VMs or physical). Some nodes run the **control plane** (the brain: API server, scheduler, controller manager, etcd). The rest are **worker nodes** that run your workloads. The kubelet on each worker talks to the API server; nothing bypasses the API server.

### Pod — the smallest unit
A **Pod** is one or more tightly coupled containers that share a network namespace and storage. Usually it's one container; sometimes it's an app + a sidecar (logging, service mesh proxy). Pods are **ephemeral** — they come and go, their IPs change. You almost never create a raw Pod directly; a higher-level controller creates them for you.

### ReplicaSet — N copies of a Pod, always
A **ReplicaSet** ensures that *N* pods matching a label selector are running. If one dies, it creates another. You rarely create ReplicaSets directly either — they're managed by Deployments.

### Deployment — rolling updates over ReplicaSets
A **Deployment** manages versioned rollouts. When you update the image in a Deployment, it creates a new ReplicaSet, scales up the new one while scaling down the old one (rolling update), and keeps history for rollback. It's the standard way to run stateless workloads.

### Service — stable network identity for ephemeral pods
Pod IPs change; Services don't. A **Service** is a stable virtual IP + DNS name that load-balances to any pod matching its selector. Three common types: **ClusterIP** (internal only, default), **NodePort** (exposed on a fixed port of every node — rarely used in prod), **LoadBalancer** (cloud-provisioned external IP). In-cluster, apps talk to `http://billing.default.svc.cluster.local` regardless of which pods are alive.

### Ingress — HTTP routing at the edge
A **Service of type LoadBalancer** gives you one external IP per service, which gets expensive. An **Ingress** is an HTTP(S) reverse proxy that routes many hostnames and paths to different services behind a single LoadBalancer. Requires an **Ingress controller** (nginx-ingress, Traefik, AWS ALB controller) to actually process the rules. TLS termination usually lives here.

### ConfigMap vs Secret
**ConfigMap** = non-sensitive config (feature flags, URLs, limits). **Secret** = sensitive data (passwords, tokens, certs). Both mount into pods as environment variables or files. Crucially, **Secrets are base64-encoded, not encrypted** — encryption at rest is optional and must be configured. For real secrets in production you want external secret managers (Vault, AWS Secrets Manager, sealed-secrets).

### Namespaces — logical isolation
A **namespace** is a scope for names — you can have a `billing` service in `dev` and another `billing` in `prod`. Namespaces carry RBAC, resource quotas, and network policies. They're the "teams and environments" boundary inside a shared cluster.

### Labels and selectors — the loose-coupling mechanism
Every object has **labels** (key=value tags). Controllers find their pods via **label selectors**. A Deployment says "manage any pod with label `app=web, env=prod`". A Service says "send traffic to any pod with label `app=web`". This is how Kubernetes stays decoupled — no hardcoded IDs, just labels.

### Declarative vs imperative
`kubectl create` and `kubectl run` are **imperative** — they do one thing. `kubectl apply -f` is **declarative** — it takes your YAML as the source of truth and reconciles the cluster toward it. Production workflows are always declarative (stored in Git, applied via CI).

### StatefulSet — when pods need identity
Deployments treat pods as interchangeable. Databases can't be — pod-0 and pod-1 have different roles (primary/replica) and must keep their storage. A **StatefulSet** gives each pod a stable name (`mysql-0`, `mysql-1`), stable storage (PersistentVolumeClaim per pod), and ordered start/stop. This is the right abstraction for stateful workloads.

### The desired-state → actual-state loop, end-to-end
When you `kubectl apply`: (1) kubectl sends YAML to the API server. (2) API server validates and stores it in etcd. (3) Controllers see the new object, create children (e.g. Deployment creates ReplicaSet creates Pods). (4) Scheduler picks a node for each pod. (5) kubelet on that node pulls the image and starts the container via containerd. (6) Each controller keeps watching for drift, forever. Every step is an event on a shared event bus.

## Common gotchas

- Creating bare Pods instead of Deployments — when the node dies, your pod is gone forever
- Forgetting resource `requests` and `limits` — one noisy pod can starve the whole node
- Using `latest` tag — when a pod restarts, it can silently pick up a different image
- Not setting liveness/readiness probes — kubelet can't tell if your app is healthy or still warming up
- Exposing `NodePort` services to the public internet — security hole, use LoadBalancer/Ingress instead
- Storing secrets in ConfigMaps because "it's easier" — they end up in logs, backups, terraform state
- Over-using namespaces as security boundaries — they're isolation by default only for names, not for network or RBAC (you configure those)

## Interview questions with model answers

### Q: What happens when you run `kubectl apply -f deployment.yaml`?
**Short answer:** kubectl sends the YAML to the API server, which validates and persists it to etcd. The Deployment controller notices the new object, creates a ReplicaSet, which creates Pods. The scheduler assigns pods to nodes, and each node's kubelet starts the containers via containerd.
**Deeper:** The full flow involves roughly seven components. kubectl authenticates, converts YAML to JSON, and POSTs to the kube-apiserver. The apiserver runs admission controllers (RBAC, resource quotas, mutating/validating webhooks), then writes the final object to etcd. The Deployment controller, watching the API for Deployment changes, sees the new object and creates a ReplicaSet. The ReplicaSet controller creates pods. Each new pod starts "pending" — the scheduler picks a node based on resource requests, affinities, and taints. The kubelet on that node pulls the image and starts the container. Throughout, each controller keeps reconciling — if a pod dies, the ReplicaSet creates a new one.
**Watch out for:** Candidates skip the admission controller step. That's where resource quotas, security policies, and webhooks (like Istio injection) actually take effect.

### Q: How does a Service route traffic to the right pods?
**Short answer:** A Service has a label selector. Kubernetes continuously populates an Endpoints object with the IPs of pods matching that selector. kube-proxy on each node programs iptables (or IPVS) rules to DNAT service traffic to those endpoints.
**Deeper:** When a client inside the cluster calls `http://web.default.svc.cluster.local`, CoreDNS resolves it to the Service's ClusterIP (a virtual IP that doesn't actually belong to any machine). A packet to that IP hits iptables rules installed by kube-proxy, which rewrites the destination to one of the real pod IPs (load-balanced by random or round-robin). Pods come and go; kube-proxy and CoreDNS stay in sync via the Endpoints controller. That's why apps never know pod IPs — the Service is the stable abstraction.
**Watch out for:** Saying "the Service is a load balancer" is close but shallow. It's actually iptables rules + a virtual IP, not a real TCP proxy.

### Q: Deployment vs StatefulSet — when to use which?
**Short answer:** Deployments treat pods as identical and interchangeable — ideal for stateless apps. StatefulSets give each pod a stable identity, stable storage, and ordered lifecycle — required for stateful apps.
**Deeper:** Your web API? Deployment. Postgres with a primary and replicas? StatefulSet. The key differences: Deployments create pods with random suffix names (`web-7b9c8f-xkf9g`); StatefulSets use ordered integers (`mysql-0`, `mysql-1`). Deployments attach any available PersistentVolumeClaim; StatefulSets provision one PVC per pod via `volumeClaimTemplates` and bind pod-0's PVC to pod-0 forever. On scale-down, StatefulSets delete in reverse order (highest ordinal first), preserving pod-0 as the primary. For cache/queue systems that *can* rebuild state (Redis in non-cluster mode), Deployments are fine; for anything with durable data and role assignment, it's StatefulSet.
**Watch out for:** "StatefulSet is for databases" — true but incomplete. The real trigger is: *does each pod have a unique identity or durable storage?*

### Q: What's the difference between a ConfigMap and a Secret, and why is a Secret not really secret?
**Short answer:** Both store key-value data that pods can consume as env vars or files. Secrets carry a semantic "please handle with care" flag and are base64-encoded; they are *not* encrypted by default.
**Deeper:** A Secret's only real protection is that RBAC policies typically restrict who can read them and that they're stored in memory (tmpfs) when mounted into pods. In etcd, they sit as base64 — anyone with etcd read access sees them in seconds. Real protection requires enabling **encryption at rest** for Secrets in etcd (a cluster-level config), or using **external secret stores** (Vault, AWS Secrets Manager, GCP Secret Manager) with a sync operator like External Secrets Operator. In production, treat native Secrets as a handoff mechanism, not a security boundary.
**Watch out for:** Candidates who say "Secrets are encrypted" without qualification — that's the trap.

### Q: Explain liveness, readiness, and startup probes.
**Short answer:** **Readiness** says "I'm ready to serve traffic" (controls whether you're in the Service's endpoint list). **Liveness** says "I'm still healthy" (failure → kubelet restarts the container). **Startup** says "I'm still booting" (shields liveness from firing during slow starts).
**Deeper:** Readiness is for load balancing. If your pod is overloaded or warming a cache, failing readiness removes you from the Service, so traffic flows to healthier peers. When ready again, traffic returns. Liveness is for self-healing. If your pod is hung (deadlocked thread, leaked FDs), failing liveness gets you restarted. Startup is for apps that take a long time to boot (JVM, ML model load) — without it, liveness probes would fire during boot and kill the pod in a restart loop. Rule of thumb: always implement readiness; add liveness only if you understand what "unhealthy" means for your app; use startup for anything over 30 seconds to boot.
**Watch out for:** Using the same endpoint for liveness and readiness. If readiness fails because a dependency is slow, liveness shouldn't also fail — that just restarts the pod without fixing anything.

### Q: How do rolling updates work in a Deployment?
**Short answer:** The Deployment creates a new ReplicaSet at the new version, scales it up one pod at a time while scaling down the old ReplicaSet, respecting `maxSurge` and `maxUnavailable` parameters.
**Deeper:** Say you have 10 replicas with `maxSurge=25%` and `maxUnavailable=25%`. At any moment, the total pods can briefly be up to 12 (surge) and the ready pods can briefly dip to 7 (unavailable). Kubernetes scales the new RS up by 1 → waits for the new pod to become ready → scales the old RS down by 1 → repeats. Roll-back is free: `kubectl rollout undo` flips which ReplicaSet is being scaled. For blue-green or canary patterns, you usually reach for Argo Rollouts or Flagger instead of raw Deployments.
**Watch out for:** Missing readiness probes — without them, the rollout races ahead even when new pods aren't actually serving, causing a brief outage.

### Q: What's an Ingress, and why not just use a LoadBalancer Service per app?
**Short answer:** LoadBalancer Services cost money (one external IP/LB per service). Ingress multiplexes many hostnames and paths onto a single load balancer with HTTP routing rules.
**Deeper:** In a cloud, each LoadBalancer Service provisions a real cloud LB (AWS NLB/ALB, GCP LB), which has a monthly cost and often a long provisioning time. If you have 20 services, that's 20 LBs. An Ingress controller (nginx-ingress, Traefik, AWS ALB controller) runs as pods that receive traffic from a single LoadBalancer Service, reads the cluster's Ingress objects, and routes based on Host headers and paths. You also get TLS termination, redirects, rate limiting, and path-based A/B testing — none of which a raw LB gives you. The Gateway API is the evolving successor standard; same idea, better semantics.
**Watch out for:** Forgetting that an Ingress *resource* is inert without an Ingress *controller* running in the cluster.

## Follow-up questions interviewers love

- What is etcd's role, and what breaks first when it's unhealthy?
- How does the scheduler decide where to place a new pod?
- What's the difference between a DaemonSet and a Deployment?
- How would you do a canary deployment natively in Kubernetes?
- Explain taints and tolerations with a real-world example.
- What's a PodDisruptionBudget and when does it matter?
- How do pods talk to each other across nodes?
- What's the difference between `kubectl rollout restart` and `kubectl delete pod`?

## Build with AI — the workflow

1. *"Generate a complete Kubernetes manifest set for a Node.js app: Deployment with 3 replicas, Service of type ClusterIP, Ingress, ConfigMap for non-secret config, Secret for DB password, and a PodDisruptionBudget. Explain every field."* — Builds a reference artefact.
2. *"Walk me through what happens when a worker node in my Kubernetes cluster dies. What does Kubernetes do, in what order, and on what timeline?"* — Understands self-healing.
3. *"Here's a Deployment manifest. Review it for production readiness — probes, resources, labels, security context. List every fix needed."* — Practices code review.
4. *"Explain Kubernetes Services, Endpoints, kube-proxy, and CoreDNS in one coherent story, as if I'm preparing for a senior-level interview."* — Practises the network data path.
5. *"I'm migrating a 5-service Docker Compose app to Kubernetes. Map each Compose concept to its closest Kubernetes equivalent and point out where the mapping breaks."* — Bridges yesterday's knowledge.

## Recap

- Kubernetes = declarative control loops that make reality match your YAML.
- Pods are ephemeral; Deployments (stateless) and StatefulSets (stateful) manage them.
- Services give stable network identity; Ingress multiplexes HTTP onto one LB.
- ConfigMaps for config, Secrets for sensitive data — but Secrets aren't encrypted by default.
- Probes are essential: readiness for traffic, liveness for self-healing, startup for slow boots.
- Everything reconciles continuously. You don't run it; it runs itself against your description.
- Namespaces scope names and policies, not network by default.

> **Want an AI tutor on this exact lesson plus a full mock DevOps interview?** → **[vibelearner.com/tracks/devops/day/5](https://vibelearner.com/tracks/devops/day/5)**
