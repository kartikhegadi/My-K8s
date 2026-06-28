# Kubernetes Real-World Failures & Fixes

### A 4-Part Field Guide to Production Kubernetes Troubleshooting

*Notes by **InfraCorps***

> *"The job of a DevOps engineer doesn't end at writing YAML — it begins when that YAML fails in production."*

---

## 📘 About This Series

This series documents **40 real-world Kubernetes failure scenarios** commonly asked in DevOps interviews and encountered in production — each broken down into the exact symptom, the root cause, and the step-by-step fix.

The series is organized into four parts, progressing from core pod-level failures to advanced networking, ingress, and configuration issues.

---

## 📂 Series Index

| Set | Title | Focus Area | Topics Covered |
|:---:|-------|-------------|:---:|
| **01** | [CrashLoopBackOff & Friends](./01-crashloopbackoff-and-friends.md) | Core Pod Failures | 10 |
| **02** | [Fix Pods & Nodes](./02-fix-pods-and-nodes.md) | Init Containers, Sidecars, Scheduling & Rollouts | 10 |
| **03** | [Production Scenarios — Scaling, Rollouts & Services/DNS](./03-production-scenarios-scaling-rollouts-services-dns.md) | Scaling Limits & Services/DNS | 10 |
| **04** | [Ingress, ConfigMaps & Secrets](./04-ingress-configmaps-secrets.md) | Ingress Routing & Configuration Injection | 10 |

**Total: 40 troubleshooting scenarios across 4 sets.**

---

## 🧭 Detailed Topic Index

### 📦 Set 1 — CrashLoopBackOff & Friends
*[Open file →](./01-crashloopbackoff-and-friends.md)*

| # | Topic |
|:---:|-------|
| 1 | CrashLoopBackOff |
| 2 | ImagePullBackOff |
| 3 | The Vanishing Pod |
| 4 | Instant Parsing Failure |
| 5 | Pod Stuck in Pending |
| 6 | OOMKilled |
| 7 | Silent Restarts, No Logs |
| 8 | Exit Code 1 |
| 9 | Deployment With No Pods |
| 10 | Skipped Init Containers |

---

### 📦 Set 2 — Fix Pods & Nodes
*[Open file →](./02-fix-pods-and-nodes.md)*

| # | Topic |
|:---:|-------|
| 1 | Init Container Blocking the Main Container |
| 2 | Sidecar Can't Read the Main Container's Logs |
| 3 | Latency Between Pods on Different Nodes |
| 4 | Pod Not Getting an IP Address |
| 5 | ReplicaSet Creating Zero Pods |
| 6 | Rollback That Won't Roll Back |
| 7 | Rollout Stuck After Resume |
| 8 | Blue-Green Traffic Stuck on Blue |
| 9 | Canary Getting 100% of the Traffic |
| 10 | Image Update Ignored by Kubernetes |

---

### 📦 Set 3 — Production Scenarios: Scaling, Rollouts & Services/DNS
*[Open file →](./03-production-scenarios-scaling-rollouts-services-dns.md)*

| # | Topic |
|:---:|-------|
| 1 | Missing Replicas After Scaling |
| 2 | Downtime During a Rolling Update |
| 3 | ClusterIP Service Unreachable |
| 4 | DNS Resolves but Connection Refused |
| 5 | NodePort Works on Some Nodes, Not Others |
| 6 | NodePort Unreachable From Outside the Cluster |
| 7 | Traffic Stuck on One Node |
| 8 | Traffic Fails Only via Service Name |
| 9 | Headless Service Returns No DNS Records |
| 10 | Pods Stuck in ContainerCreating |

---

### 📦 Set 4 — Ingress, ConfigMaps & Secrets
*[Open file →](./04-ingress-configmaps-secrets.md)*

| # | Topic |
|:---:|-------|
| 1 | Ingress Returns 404 on a Working Service |
| 2 | TLS Configured but HTTP Still Works |
| 3 | Path Rewrite Not Taking Effect |
| 4 | Exact Path Match Blocking Valid Requests |
| 5 | Wrong Service Receiving the Traffic |
| 6 | ConfigMap Exists, but App Crashes on Missing Config |
| 7 | envFrom Injects Empty Environment Variables |
| 8 | Mounted Config File Turns Into a Directory |
| 9 | Secret Injected, but Credentials Are Stale |
| 10 | ConfigMap Update Not Reflected in Running Pods |

---

## 🗂️ Recommended Reading Order

```
Set 1  →  Set 2  →  Set 3  →  Set 4
Pods      Pods &     Scaling &    Ingress &
& Basics  Nodes      Services/DNS Config
```

Each set builds on concepts from the previous one. Readers already comfortable with core pod troubleshooting can jump directly to the set most relevant to their current issue using the index above.

---

**Series:** Kubernetes Real-World Failures & Fixes
**Parts:** 4
**Total Scenarios:** 40
**Maintained by:** InfraCorps
