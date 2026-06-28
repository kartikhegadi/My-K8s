# ⚙️ 10 Real-World Kubernetes Failures & Their Fixes
### 🧩 Set 3 — Production Scenarios: Scaling, Rollouts & Services/DNS
*Notes by **InfraCorps***

> *Set 3 moves into production territory — scaling limits, zero-downtime rollouts, and the many ways Services and DNS can quietly break.*

```
   ☁️  ────────────────────────────────────────  ☁️
        kubectl get svc → 🌐 DNS resolution failed
   ☁️  ────────────────────────────────────────  ☁️
```

---

## 🗺️ What's Inside

| # | Failure | One-line Symptom |
|---|---------|-------------------|
| [1](#1--missing-replicas-after-scaling) | 📉 Missing Replicas After Scaling | Asked for 6, only 4 show up |
| [2](#2--downtime-during-a-rolling-update) | ⏱️ Downtime During a Rolling Update | Users notice gaps during deploys |
| [3](#3--clusterip-service-unreachable) | 🔌 ClusterIP Service Unreachable | Service exists, but nothing connects |
| [4](#4--dns-resolves-but-connection-refused) | 🚪 DNS Resolves but Connection Refused | Right service, wrong port |
| [5](#5--nodeport-works-on-some-nodes-not-others) | 🧱 NodePort Works on Some Nodes, Not Others | One node blocks it, others don't |
| [6](#6--nodeport-unreachable-from-outside-the-cluster) | 🌍 NodePort Unreachable From Outside the Cluster | Internal access fine, external fails |
| [7](#7--traffic-stuck-on-one-node) | ⚖️ Traffic Stuck on One Node | Endpoints look fine, routing doesn't |
| [8](#8--traffic-fails-only-via-service-name) | 🏷️ Traffic Fails Only via Service Name | Pod IP works, service name doesn't |
| [9](#9--headless-service-returns-no-dns-records) | 👻 Headless Service Returns No DNS Records | No records despite running pods |
| [10](#10--pods-stuck-in-containercreating) | 📦 Pods Stuck in ContainerCreating | Replicas ready, but stuck mounting |

---

## 1. 📉 Missing Replicas After Scaling

`Desired: 6` → `Running: 4` → `⏳ 2 Pending` → `🚫 Failed Scheduling (memory)`

**❓ The question:**
> You scaled your deployment from two to six replicas, but only four pods are running even though there are enough nodes. How would you investigate the reason for the missing replicas?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get deployment` | Only **4 pods available**, 2 replicas missing/pending |
| 2 | `kubectl describe replicaset <name>` | ReplicaSet exists but failed to create 2 pods — **FailedCreate** error tied to resource quota/limits |
| 3 | `kubectl get pods` | 4 running, **2 stuck in Pending** |
| 4 | `kubectl describe pod <pending-pod>` | **FailedScheduling** — nodes don't have enough free memory |
| 5 | `kubectl describe nodes` | Nodes are Ready, but **memory is nearly full** |
| 6 | Reduce memory `requests` in the deployment spec, or add node capacity | Pending pods can now be scheduled |
| 7 | `kubectl apply -f deployment.yaml` | All 6 replicas reach **Running** — scaling issue resolved |

> 💡 **Takeaway:** "Enough nodes" doesn't mean "enough *resources*." Check memory/CPU headroom, not just node count.

---

## 2. ⏱️ Downtime During a Rolling Update

**❓ The question:**
> You've performed a rolling update, but users reported downtime during the rollout. How would you identify what went wrong and ensure zero downtime in future rolling updates?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl rollout status deployment/<name>` | Rollout wasn't smooth — new pods took time to become ready, possibly causing the downtime |
| 2 | Review the deployment strategy | `maxUnavailable: 2` — allowed **2 pods to go down at once**, causing service unavailability |
| 3 | Inspect readiness probes | No initial delay before checking readiness — pods marked **ready too soon**, before actually able to serve traffic |

**The fix — update the rollout strategy:**

| Setting | New Value | Why |
|---------|-----------|-----|
| `maxUnavailable` | `0` | No existing pods are taken down until new ones are ready |
| `maxSurge` | `1` | One extra pod above desired count during rollout, for a smooth transition |
| `readinessProbe.initialDelaySeconds` | `10` | Gives the app time to actually start before being marked ready |

> 💡 **Takeaway:** Zero downtime is a combination of **two settings**: a strict rollout strategy (`maxUnavailable: 0`) and an honest readiness probe (`initialDelaySeconds`).

---

## 3. 🔌 ClusterIP Service Unreachable

```
  Service ✅ Created   →   🔍 Endpoints: NONE   →   ❌ nothing to forward traffic to
```

**❓ The question:**
> After creating a ClusterIP service, your pods are running but the service IP is unreachable from inside the cluster. What would you check?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get svc` | Service exists, but IP is unreachable |
| 2 | `kubectl get endpoints <service-name>` | **No endpoints** — service has nothing to forward traffic to |
| 3 | Compare service `selector` vs pod `labels` | **Selector doesn't match** the pod labels |
| 4 | Fix the selector in the service YAML | Service can now find the pods |
| 5 | Verify connectivity | Service IP is reachable again — resolved |

> 💡 **Takeaway:** Zero endpoints is *always* a selector/label mismatch (or no pods matching it) — check `kubectl get endpoints` before anything else.

---

## 4. 🚪 DNS Resolves but Connection Refused

**❓ The question:**
> Your ClusterIP service resolves in DNS, but the request fails with "connection refused." What could have caused this?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get svc` | Service is created and reachable by DNS |
| 2 | `kubectl get endpoints <service-name>` | Endpoint exists — the selector **is** working |
| 3 | Check `port` vs `targetPort` | Service forwards to **port 80**, but the container actually listens on **port 8080** — mismatch |
| 4 | Fix `targetPort`: `80` → `8080` | Traffic now lands on the port the app is actually listening on |
| 5 | `kubectl get svc` again | Service forwards correctly, requests succeed |

> 💡 **Takeaway:** "Connection refused" with working DNS = almost always a `port` vs `targetPort` mismatch.

---

## 5. 🧱 NodePort Works on Some Nodes, Not Others

```
  Node 1: ❌ connection refused     Node 2: ✅ OK     Node 3: ✅ OK
        (port 30080 blocked by a firewall deny rule)
```

**❓ The question:**
> Your NodePort service works on one node but fails on the others. How would you troubleshoot this?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get svc` | NodePort service correctly created, exposing **30080** on every node |
| 2 | `kubectl get endpoints` | Backing pods present and healthy — selector and internal routing are fine |
| 3 | Test the NodePort from each node | Node 2 & Node 3 respond normally; **Node 1 returns connection refused** — kube-proxy forwarding isn't happening there |
| 4 | Inspect firewall rules on Node 1 | Port **30080 is explicitly blocked** by a deny rule |
| 5 | Remove the deny rule, allow port 30080 | NodePort now works on Node 1 too — firewall was the root cause |

> 💡 **Takeaway:** When a NodePort works on *some* nodes but not others, the problem is almost never Kubernetes — check per-node firewall rules first.

---

## 6. 🌍 NodePort Unreachable From Outside the Cluster

**❓ The question:**
> Clients outside the cluster cannot access your NodePort, even though the required port is open. What would you investigate?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get svc` | Correctly created as NodePort, exposing the required port |
| 2 | `kubectl get endpoints` | Backend pod IPs present and healthy — not a pod readiness or selector issue |
| 3 | Test the NodePort **inside** the cluster | Responds correctly — confirms the issue is outside Kubernetes |
| 4 | Test from an **external network** using the node's public IP | Access fails — packets aren't reaching the node at all |
| 5 | Check external/node-level firewall | Port is **blocked at the network/firewall level**, outside Kubernetes' control |
| 6 | Escalate to the firewall/network team to allow the NodePort externally | Once opened, service becomes accessible |

> 💡 **Takeaway:** Internal access working + external access failing = look outside the cluster, at network/firewall rules, not at Kubernetes config.

---

## 7. ⚖️ Traffic Stuck on One Node

**❓ The question:**
> Your service shows multiple endpoints, but traffic consistently reaches only pods on a particular node. What node-level issue could cause this imbalance?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get endpoints <service-name>` | Multiple endpoints listed — Kubernetes knows about all the pods |
| 2 | Verify pod-to-node distribution | Pods are spread across nodes, so all nodes *should* receive traffic |
| 3 | Check kube-proxy status | **kube-proxy is not running on Node 2** — traffic only reaches pods on Node 1 |
| 4 | Inspect kube-proxy logs | Shows it **can't set iptables rules**, blocking traffic on Node 2 |
| 5 | Delete the broken kube-proxy pod on Node 2 | Since kube-proxy runs as a **DaemonSet**, Kubernetes automatically recreates it |
| 6 | Verify traffic balancing | Traffic now reaches pods across multiple nodes — kube-proxy failure confirmed as root cause |

> 💡 **Takeaway:** Uneven traffic across otherwise-healthy endpoints points straight at kube-proxy health on the under-served node.

---

## 8. 🏷️ Traffic Fails Only via Service Name

```
  curl <pod-IP>        →   ✅ works
  curl <service-name>  →   ❌ fails   (CoreDNS in CrashLoopBackOff)
```

**❓ The question:**
> Your service exists and has correct endpoints, but traffic fails only when using the service name. How do we find the root cause?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get svc` / `kubectl get endpoints` | Both exist — service *should* be able to route traffic |
| 2 | Test direct pod IP connectivity | **Works fine** — application and pod networking are healthy |
| 3 | Test DNS resolution for the service name | **Fails** — points to a cluster DNS problem, not the service or app |
| 4 | `kubectl get pods -n kube-system` / check CoreDNS logs | **CoreDNS is in CrashLoopBackOff** — the actual root cause |
| 5 | Restart CoreDNS | DNS resolution is restored, service traffic succeeds |

> 💡 **Takeaway:** If pod IP works but the service *name* doesn't, the problem lives in CoreDNS — not the service object itself.

---

## 9. 👻 Headless Service Returns No DNS Records

**❓ The question:**
> Your headless service returns no DNS records even though pods are running. How will you troubleshoot this?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | Check the service config | `clusterIP: None` confirms it's a headless service — DNS should return pod IPs if endpoints exist |
| 2 | Test DNS resolution | **No DNS record returned** — Kubernetes DNS has no endpoint to publish |
| 3 | `kubectl get endpoints <service-name>` | **No endpoints found**, even though pods are running |
| 4 | Compare service YAML vs pod YAML | Service selector uses `backend`, but pod label is **`back-end`** — mismatch |
| 5 | Fix the selector to match the pod label | Endpoints are created immediately, DNS records are restored |

> 💡 **Takeaway:** Headless services follow the exact same selector/label rules as ClusterIP services — a tiny typo (`backend` vs `back-end`) silently breaks DNS entirely.

---

## 10. 📦 Pods Stuck in ContainerCreating

```
  Replicas: 3   Ready: 1   ContainerCreating: 2   →   📁 hostPath volume missing on node
```

**❓ The question:**
> Your deployment shows the desired number of replicas, but some pods never become ready and stay in ContainerCreating for a long time. How would you troubleshoot the cause and fix it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get deployment` / `kubectl get pods` | 3 replicas desired, only **1 ready** — others stuck in **ContainerCreating** |
| 2 | `kubectl describe pod <pod-name>` | Pod can't start — **volume mount failed** |
| 3 | Inspect the pod spec's volume section | Expects `/data/app` to exist as a **hostPath directory on the node** — but it doesn't exist |
| 4 | Create the directory on the node with correct permissions, or fix the path in the pod YAML | Removes the blocker |
| 5 | `kubectl get pods` | Pods move from ContainerCreating → Running and Ready — deployment now meets desired replica count |

> 💡 **Takeaway:** Long-stuck ContainerCreating almost always traces back to image pulls or volume mounts — `hostPath` volumes are only as reliable as the directory actually existing on that specific node.

---

```
   ✅ ────────────────────────────────────  ✅
              End of Set 3
   ✅ ────────────────────────────────────  ✅
```

<sub>Notes by **InfraCorps** — Set 3 of the Kubernetes Real-World Failures series.</sub>
