# ⚙️ 10 Real-World Kubernetes Failures & Their Fixes
### 🧩 Set 1 — CrashLoopBackOff & Friends
*Notes by **InfraCorps***

> *A field guide to the Kubernetes failures every DevOps engineer eventually meets — what breaks, why it breaks, and the exact commands to fix it.*

```
   ☁️  ────────────────────────────────────────  ☁️
        kubectl get pods → 🔴 CrashLoopBackOff
   ☁️  ────────────────────────────────────────  ☁️
```

---

## 🗺️ What's Inside

| # | Failure | One-line Symptom |
|---|---------|-------------------|
| [1](#1--crashloopbackoff) | 🔁 CrashLoopBackOff | Pod restarts endlessly |
| [2](#2--imagepullbackoff) | 🖼️ ImagePullBackOff | Pod can't fetch its image |
| [3](#3--the-vanishing-pod) | 👻 The Vanishing Pod | `apply` succeeds, no pod appears |
| [4](#4--instant-parsing-failure) | 📄 Instant Parsing Failure | `apply` fails before reaching the cluster |
| [5](#5--pod-stuck-pending) | ⏳ Pod Stuck in Pending | No errors, no logs, just stuck |
| [6](#6--oomkilled) | 💥 OOMKilled | Pod killed for using too much memory |
| [7](#7--silent-restarts-no-logs) | 🤐 Silent Restarts, No Logs | Restarting, but nothing to read |
| [8](#8--exit-code-1) | ❌ Exit Code 1 | Container dies instantly |
| [9](#9--deployment-with-no-pods) | 🚫 Deployment With No Pods | Deployment exists, pods don't |
| [10](#10--skipped-init-containers) | ⏭️ Skipped Init Containers | Init logic silently ignored |

---

## 1. 🔁 CrashLoopBackOff

`Pod Starts` → `💥 Crashes` → `⏱️ Backs Off & Retries` → *(loops back to start)*

**❓ The question:**
> One of the most asked Kubernetes interview questions — your application pod keeps going into CrashLoopBackOff. How would you troubleshoot it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get pods` | Confirms which pod is in `CrashLoopBackOff` |
| 2 | `kubectl describe pod <pod-name>` | Shows the exact stage of failure (here: failed *after* the image was pulled, during restart) |
| 3 | `kubectl logs <pod-name>` | Reveals the real error — in this case, a **missing environment variable**: `DB_HOST` |
| 4 | Edit the YAML | Add the missing environment variable |
| 5 | `kubectl apply -f <file>.yaml` | Redeploy with the fix |

> 💡 **Takeaway:** `describe` tells you *where* it failed. `logs` tells you *why*.

---

## 2. 🖼️ ImagePullBackOff

**❓ The question:**
> The second most asked DevOps interview question — a pod fails with ImagePullBackOff. What could be the reason, and how would you solve it?

**✅ The answer, step by step:**

| Step | Command | What It Tells You |
|------|---------|--------------------|
| 1 | `kubectl get pods` | Identifies the failing pod |
| 2 | `kubectl describe pod <pod-name>` | Shows the exact step where the pull failed |

**Most common root cause:** 🏷️ the image tag simply doesn't exist in the registry.

**Fix:** use a different available image tag, or re-run the CI pipeline to rebuild and push the correct image.

---

## 3. 👻 The Vanishing Pod

```
$ kubectl apply -f pod.yaml
pod/my-app configured ✓        ...but → 0 pods running 😶
```

**❓ The question:**
> You applied a Kubernetes YAML but no pod got created — even though the CLI says it was configured or created. How would you troubleshoot this?

**✅ The answer, step by step:**

| Step | Action | Why |
|------|--------|-----|
| 1 | Open the YAML and check `kind:` | A typo here (e.g. `Pdo` instead of `Pod`) silently creates the wrong / no resource |
| 2 | Check `apiVersion` and `spec` | Every field must be correct for the resource to be created |
| 3 | `kubectl get all` | Shows everything that actually got created — deployments, services, pods |
| 4 | Run a dry run (`--dry-run=client`) | Catches formatting mistakes *before* you reapply |

---

## 4. 📄 Instant Parsing Failure

**❓ The question:**
> You ran `kubectl apply` and got a parsing error immediately — before it even hit the cluster. What went wrong?

**✅ The answer:**

If `kubectl apply` fails **instantly**, Kubernetes never even saw your file 🚧. The problem is 100% in the YAML itself, not the cluster. Check these four things:

| # | Check For |
|---|-----------|
| 1️⃣ | Indentation error |
| 2️⃣ | Invalid syntax |
| 3️⃣ | Wrong data type |
| 4️⃣ | Run a YAML lint |

> 💡 **Takeaway:** A parsing error = the file was broken from the start. Lint before you apply.

---

## 5. ⏳ Pod Stuck in Pending

```
  😶 No containers   →   [ ⏳ PENDING ]   →   🚫 Can't find a node
  No errors, no logs                          to schedule on
```

**❓ The question:**
> You deployed your pod, but it's stuck in the Pending state — no containers, no errors, no logs. What's blocking it from getting scheduled?

**✅ The answer, step by step:**

| Step | Action | What to Look For |
|------|--------|-------------------|
| 1 | `kubectl describe pod <pod-name>` | Events explaining why scheduling failed |
| 2 | Check resource requests | Insufficient CPU/memory — update them in the pod spec if so |
| 3 | Check taints, affinity, node selectors | A scheduling rule may be excluding every available node |

> 💡 **Takeaway:** Pending almost always means the scheduler can't find a node that satisfies every rule you've set.

---

## 6. 💥 OOMKilled

**❓ The question:**
> A Python application pod is killed repeatedly. What's the root cause and fix?

**✅ The answer:**

**OOMKilled** = Out Of Memory Killed 🧠💥. The container used more memory than Kubernetes allocated to it.

| Step | Command | Result |
|------|---------|--------|
| 1 | `kubectl describe pod <pod-name>` | Confirms termination reason: OOM kill, **exit code 137** |
| 2 | `kubectl top pod <pod-name>` | Shows live usage — here, **75% CPU**, **540Mi memory** |
| 3 | Check the deployment YAML | Memory limit was set to **512Mi** — too low for actual usage |
| 4 | Edit the limit | Raise it from `512Mi` → `768Mi` |
| 5 | `kubectl apply -f deployment.yaml` | Redeploy |
| 6 | `kubectl describe pod` again | No OOM error this time — fixed ✅ |

> 💡 **Takeaway:** Exit code `137` is your signature for OOMKilled. Match the memory *limit* to real usage, not a guess.

---

## 7. 🤐 Silent Restarts, No Logs

**❓ The question:**
> *(Debugging under 1 minute)* A pod restarts frequently but no logs are visible. How do you debug this?

**✅ The answer, step by step:**

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `kubectl get pod` | Confirms restart count is climbing (e.g. restarts: 6, status: CrashLoopBackOff) |
| 2 | `kubectl describe pod <pod-name>` | Look in the Events section for clues |
| 3 | `kubectl logs --previous <pod-name>` | Pulls logs from the **last failed run** — current logs are empty, but the previous run isn't |
| 4 | Read the error | Example: `database not reachable` |
| 5 | Check the database service | Confirm it exists and is running in the cluster |
| 6 | Hand off if needed | If the DB is up but still unreachable, escalate to the database team for connectivity/auth checks |
| 7 | Redeploy & verify | Confirm the pod runs without restarting |

> 💡 **Takeaway:** `--previous` is the key flag — it's the only way to see logs from a container that's already gone.

---

## 8. ❌ Exit Code 1

**❓ The question:**
> *(Debugging under 1 minute)* Your container exits immediately with exit code 1. How do you find and fix the failure?

**✅ The answer, step by step:**

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `kubectl get pod <pod-name>` | Confirm the error / CrashLoopBackOff state |
| 2 | `kubectl describe pod <pod-name>` | Check Events for messages about the container exit |
| 3 | `kubectl logs --previous <pod-name>` | Capture logs from the last failed run |
| 4 | Read the error | E.g. "configuration file not found" or "invalid environment variable value" |
| 5 | Update the ConfigMap | Add the missing environment variable |
| 6 | `kubectl apply -f deployment.yaml` | Redeploy with the fix |
| 7 | `kubectl get pod <pod-name>` | Confirm it's running |

> 💡 **Takeaway:** Exit code `1` = a general application error. The logs from the *previous* run almost always name the exact missing piece.

---

## 9. 🚫 Deployment With No Pods

```
  ✅ Deployment Created   →   🚫 0 Pods Running   ⟶   🔐 missing pull secret
```

**❓ The question:**
> Your deployment was created but no pod started — the YAML looks fine. What could be missing?

**✅ The answer, step by step:**

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `kubectl get deployment` | Confirm the deployment exists |
| 2 | `kubectl get pods` | Check if any pods were created at all |
| 3 | `kubectl describe deployment <name>` | If pods are missing/pending, check Events for `failed to pull image` or `image pull secret not found` |

**If it's a missing pull secret**, create one:

```bash
kubectl create secret docker-registry <secret-name> \
  --docker-server=<registry-server> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>
```

| Line | What It Does |
|------|---------------|
| `create secret docker-registry <name>` | Creates the Kubernetes secret of that name |
| `--docker-server` | Registry server address |
| `--docker-username` / `--docker-password` | Login credentials for the registry |
| `--docker-email` | Email tied to the registry account |

Then link it in the deployment YAML under `imagePullSecrets`, and redeploy:

```bash
kubectl apply -f deployment.yaml
```

Once the secret is in place 🔑, Kubernetes pulls the image, creates the pod, and starts the container.

---

## 10. ⏭️ Skipped Init Containers

```
  ❌ WRONG                          ✅ CORRECT
  spec:                            spec:
    containers:                      initContainers: [...]
      initContainers: [...]   →      containers:
      - name: app                      - name: app
```

**❓ The question:**
> A pod is created, but its init containers are skipped. What YAML mistake might cause this?

**✅ The answer:**

Init containers always run **before** the main containers ⏭️ — but if the YAML structure is wrong, Kubernetes simply ignores them.

**The common mistake:** placing `initContainers` *nested inside* `containers`, instead of as a **sibling** at the same level under `spec`.

| Step | Action |
|------|--------|
| 1 | `kubectl describe pod <pod-name>` — confirm init containers never started and aren't listed in Events |
| 2 | Check the pod YAML under `spec` |
| 3 | Fix: define `initContainers` at the same level as `containers`, before it |
| 4 | Redeploy the corrected YAML |

> 💡 **Takeaway:** `initContainers` and `containers` are siblings under `spec` — never nest one inside the other.

---

```
   ✅ ────────────────────────────────────  ✅
              End of Set 1
   ✅ ────────────────────────────────────  ✅
```

<sub>Notes by **InfraCorps** — Set 1 of the Kubernetes Real-World Failures series.</sub>