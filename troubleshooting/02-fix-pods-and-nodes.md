# ⚙️ 10 Real-World Kubernetes Failures & Their Fixes
### 🧩 Set 2 — Fix Pods & Nodes
*Notes by **InfraCorps***

> *Picking up where Set 1 left off — this time digging into init containers, sidecars, scheduling, networking, ReplicaSets, and rollouts.*

```
   ☁️  ────────────────────────────────────────  ☁️
        kubectl get pods → 🔁 Init:CrashLoopBackOff
   ☁️  ────────────────────────────────────────  ☁️
```

---

## 🗺️ What's Inside

| # | Failure | One-line Symptom |
|---|---------|-------------------|
| [1](#1--init-container-blocking-the-main-container) | 🚧 Init Container Blocking the Main Container | Main container never starts |
| [2](#2--sidecar-cant-read-the-main-containers-logs) | 📂 Sidecar Can't Read the Main Container's Logs | Sidecar crashes, can't find log file |
| [3](#3--latency-between-pods-on-different-nodes) | 🐢 Latency Between Pods on Different Nodes | Pods land on different nodes, causing lag |
| [4](#4--pod-not-getting-an-ip-address) | 🌐 Pod Not Getting an IP Address | Pod can't communicate at all |
| [5](#5--replicaset-creating-zero-pods) | 0️⃣ ReplicaSet Creating Zero Pods | Desired 3, current 0 |
| [6](#6--rollback-that-wont-roll-back) | ⏪ Rollback That Won't Roll Back | No previous version to revert to |
| [7](#7--rollout-stuck-after-resume) | ⏸️ Rollout Stuck After Resume | Paused deployment won't continue |
| [8](#8--blue-green-traffic-stuck-on-blue) | 🔵🟢 Blue-Green Traffic Stuck on Blue | Green pods exist, traffic ignores them |
| [9](#9--canary-getting-100-of-the-traffic) | 🐤 Canary Getting 100% of the Traffic | Canary should get a slice, not the whole pie |
| [10](#10--image-update-ignored-by-kubernetes) | 🖼️ Image Update Ignored by Kubernetes | New tag pushed, no rollout triggered |

---

## 1. 🚧 Init Container Blocking the Main Container

`Init Container Runs` → `❌ Fails (exit 1)` → `🔁 Kubernetes Retries` → *(main container never starts)*

**❓ The question:**
> You deployed a pod with an init container that runs a database migration script, and after deployment the main container never starts. Let's troubleshoot and fix it.

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get pods` | Pod is stuck in **Init:CrashLoopBackOff** — the init container is failing repeatedly, so the main container can't start |
| 2 | `kubectl describe pod <pod-name>` | Event log confirms `migrate-db` failed with **exit code 1** |
| 3 | `kubectl logs <pod-name> -c <init-container-name>` | The migration script **can't connect to the database** |
| 4 | `kubectl get svc` | The database service exists and DNS resolution works — so the DB likely isn't *ready* yet, or credentials are wrong |
| 5 | Fix DB script/credentials → `kubectl delete pod <pod-name>` → `kubectl apply -f deployment.yaml` | Redeploy with the fix |

> 💡 **Takeaway:** Kubernetes will **never** start the main container until every init container succeeds — check init container logs with `-c <name>`, not just the pod-level logs.

---

## 2. 📂 Sidecar Can't Read the Main Container's Logs

**❓ The question:**
> You create a multi-container pod using a sidecar pattern for logging, but the sidecar container is unable to read the logs written by the main container. How would you troubleshoot it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get pods` | Pod has 2 containers, but is in **CrashLoopBackOff** |
| 2 | `kubectl describe pod <pod-name>` | Both containers share the same `emptyDir` volume — but it's **mounted at different paths** in each container |
| 3 | `kubectl logs <pod-name> -c <container-name>` | The sidecar is trying to read a log file the main container hasn't created at that path — the file doesn't exist there |
| 4 | Fix the YAML | Update both containers to mount the shared volume at the **same path** |
| 5 | Delete & reapply the manifest | Pods come up running with the fix |

> 💡 **Takeaway:** A shared `emptyDir` volume only works if every container mounts it at the **exact same `mountPath`**.

---

## 3. 🐢 Latency Between Pods on Different Nodes

```
  Node A: 🧹 pre-processing pod      Node B: 🏋️ training pod
        (fully utilized — no room left on Node A)
```

**❓ The question:**
> What if there's latency when your training pods try to access pre-processing pods? How will you troubleshoot it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get pods -o wide -n ml-training` | The pre-processing pod runs on **Node A**, the training pod on **Node B** |
| 2 | Inspect pod affinity config | The rule uses `preferredDuringScheduling...` (a **soft** rule) — it *tries* to co-locate, but falls back to any available node if it can't |
| 3 | Check node resource availability | **Node A was fully utilized** — no room for the second pod, so it landed on Node B instead |
| 4 | Update the rule from `preferred` → `required` | Forces co-location on the same node |
| 5 | Reapply the manifest | Pods now scheduled together, latency resolved |

> 💡 **Takeaway:** `preferred` affinity is a *suggestion*, not a guarantee. Use `required` when co-location is non-negotiable.

---

## 4. 🌐 Pod Not Getting an IP Address

**❓ The question:**
> You notice a pod in your cluster is not receiving an IP address and cannot communicate with other pods. How would you troubleshoot it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get pods -o wide` | `web-0` pod shows **`<none>`** for IP — it was never assigned a network address |
| 2 | `kubectl describe pod web-0` | Events show **network setup failed** |
| 3 | Check CNI plugin pods | One **Calico** pod is failing — that's the likely root cause |
| 4 | Restart the failing CNI pod | CNI pod comes back online, networking is restored |
| 5 | Restart & verify `web-0` | Pod now has an IP assigned, communication works again |

> 💡 **Takeaway:** No IP on a pod almost always traces back to the CNI layer — check your CNI plugin pods (Calico, Flannel, Cilium, etc.) before anything else.

---

## 5. 0️⃣ ReplicaSet Creating Zero Pods

```
  Desired: 3   Current: 0   →   🔍 missing ServiceAccount "frontend-sa"
```

**❓ The question:**
> You created a new ReplicaSet but it isn't launching any pods even though the YAML seems correct. What could be the reason, and how would you troubleshoot it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get replicaset` | Desired replicas: **3**, current: **0** — no pods created |
| 2 | `kubectl describe replicaset <name>` | Events show pod creation failed because of a **ServiceAccount** |
| 3 | Check the pod template `serviceAccountName` | Expected a service account named **`frontend-sa`** |
| 4 | `kubectl get serviceaccount frontend-sa -n web-app` | Confirms it **doesn't exist** |
| 5 | `kubectl create serviceaccount frontend-sa -n web-app` | Creates the missing service account |

> 💡 **Takeaway:** A ReplicaSet referencing a non-existent ServiceAccount will silently fail to create *any* pods — always check `describe` Events, not just the YAML.

---

## 6. ⏪ Rollback That Won't Roll Back

**❓ The question:**
> You attempted a rollback after a failed deployment, but the pods didn't revert to the previous version. What could be preventing the rollback from taking effect?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl rollout undo deployment/<name>` | Rollback **fails** — no previous version available to revert to |
| 2 | Check `revisionHistoryLimit` | It's set to **`0`** — old ReplicaSets get deleted immediately, so there's nothing to roll back to |
| 3 | Manually revert the image | Update the deployment image to the previous known-good version |
| 4 | `kubectl rollout status deployment/<name>` | Pods now running on the previous version — resolved |
| 5 | Update `revisionHistoryLimit` going forward | Ensures future rollbacks actually have history to use |

> 💡 **Takeaway:** `revisionHistoryLimit: 0` means "no safety net." Keep at least a few revisions if rollback matters to you.

---

## 7. ⏸️ Rollout Stuck After Resume

**❓ The question:**
> You paused a rollout for some reason, but even after resuming, the deployment remains stuck in the paused state. How would you fix this and continue the rollout?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl rollout status deployment/<name>` | Rollout is **incomplete**, deployment appears stuck |
| 2 | `kubectl describe deployment <name>` | Not paused anymore, but rollout failed with **"progress deadline exceeded"** |
| 3 | Inspect the ReplicaSets | The **old ReplicaSet isn't fully scaled down** — rollout stuck midway |
| 4 | `kubectl rollout restart deployment/<name>` | Triggers a fresh rollout, forcing pods to recreate and proceed |
| 5 | `kubectl rollout status deployment/<name>` | Rollout now completes successfully |

> 💡 **Takeaway:** "Resumed" doesn't always mean "unstuck" — a mismatched ReplicaSet scale-down needs a `rollout restart`, not just `resume`.

---

## 8. 🔵🟢 Blue-Green Traffic Stuck on Blue

```
  Service Selector: env=blue  →  🔵 old pods   (❌ traffic stuck here)
                                  🟢 new pods   (✅ should get traffic)
```

**❓ The question:**
> You performed a manual blue-green deployment, but after deployment, traffic is still being routed to the old pods. What might have gone wrong?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl describe svc <service-name>` | Service exists, but need to confirm what it's targeting |
| 2 | Inspect the service `selector` | Still points to `environment: blue` — that's why traffic stays on the old version |
| 3 | Verify pod labels | New `green` pods exist and are labeled correctly — only the **service selector** is outdated |
| 4 | Update the service selector to `environment: green` | Repoints traffic to the new pods |
| 5 | `kubectl get endpoints <service-name>` | Endpoints now show green pods — traffic correctly routed |

> 💡 **Takeaway:** Blue-green cutover lives entirely in the **service selector**. The pods can be perfect and traffic still won't move until the selector changes.

---

## 9. 🐤 Canary Getting 100% of the Traffic

**❓ The question:**
> You did a canary deployment for limited testing, but the canary pods are receiving all the traffic instead of a small percentage. How would you correct this behavior?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get replicaset` | **Canary has 5 replicas** (should be 1), **stable has 0** (should be 4) — the YAML values are flipped |
| 2 | Fix the canary deployment YAML | Correct replica count from `5` → `1` |
| 3 | Fix the stable deployment YAML | Correct replica count from `0` → `4` |
| 4 | `kubectl apply -f` both YAMLs | Apply the corrected replica counts |
| 5 | Confirm traffic split | **4 stable + 1 canary** → traffic now correctly split **80% stable / 20% canary** |

> 💡 **Takeaway:** With label-based traffic splitting, the *replica count ratio* between canary and stable **is** the traffic ratio — no separate routing config needed.

---

## 10. 🖼️ Image Update Ignored by Kubernetes

```
  image: myapp:v2   +   imagePullPolicy: IfNotPresent   →   🗄️ cached image reused, no rollout
```

**❓ The question:**
> You updated the image tag in your deployment YAML, but Kubernetes didn't create new pods or trigger a rollout. What could cause the image update to be ignored?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl rollout status deployment/<name>` | Deployment is stable, but **no new rollout triggered** despite the image update |
| 2 | Check the image field in the spec | Confirms it's updated to **`v2`** — but rollout still isn't happening |
| 3 | Check `imagePullPolicy` | Set to **`IfNotPresent`** — Kubernetes may reuse the cached image instead of pulling the new one |
| 4 | Update `imagePullPolicy` to **`Always`** | Forces Kubernetes to fetch the latest image on every rollout |
| 5 | `kubectl rollout status deployment/<name>` | New pods created with the image update, rollout triggered successfully |

> 💡 **Takeaway:** If a tag-only change doesn't trigger a rollout, suspect `imagePullPolicy: IfNotPresent` — it's the most common silent cause.

---

```
   ✅ ────────────────────────────────────  ✅
              End of Set 2
   ✅ ────────────────────────────────────  ✅
```

<sub>Notes by **InfraCorps** — Set 2 of the Kubernetes Real-World Failures series.</sub>
