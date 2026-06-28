# ⚙️ 10 Real-World Kubernetes Failures & Their Fixes
### 🧩 Set 4 — Ingress, ConfigMaps & Secrets
*Notes by **InfraCorps***

> *The final set — where requests get lost in Ingress routing rules, TLS quietly fails to enforce itself, and ConfigMaps/Secrets silently feed the wrong data into running pods.*

```
   ☁️  ────────────────────────────────────────  ☁️
        curl example.com/app → 🔴 404 Not Found
   ☁️  ────────────────────────────────────────  ☁️
```

---

## 🗺️ What's Inside

| # | Failure | One-line Symptom |
|---|---------|-------------------|
| [1](#1--ingress-returns-404-on-a-working-service) | 🚫 Ingress Returns 404 on a Working Service | Path mismatch between client and rule |
| [2](#2--tls-configured-but-http-still-works) | 🔓 TLS Configured but HTTP Still Works | HTTPS enabled, but not enforced |
| [3](#3--path-rewrite-not-taking-effect) | ✂️ Path Rewrite Not Taking Effect | Backend gets the wrong path |
| [4](#4--exact-path-match-blocking-valid-requests) | 🎯 Exact Path Match Blocking Valid Requests | `/app` ≠ `/application` |
| [5](#5--wrong-service-receiving-the-traffic) | 🔀 Wrong Service Receiving the Traffic | Rule order routes traffic to the wrong host |
| [6](#6--configmap-exists-but-app-crashes-on-missing-config) | 📋 ConfigMap Exists, but App Crashes on Missing Config | Deployment never references it |
| [7](#7--envfrom-injects-empty-environment-variables) | 🌫️ envFrom Injects Empty Environment Variables | Namespace mismatch |
| [8](#8--mounted-config-file-turns-into-a-directory) | 📁 Mounted Config File Turns Into a Directory | Missing `subPath` |
| [9](#9--secret-injected-but-credentials-are-stale) | 🔑 Secret Injected, but Credentials Are Stale | DB password changed outside Kubernetes |
| [10](#10--configmap-update-not-reflected-in-running-pods) | 🔄 ConfigMap Update Not Reflected in Running Pods | Old pods never reload env vars |

---

## 1. 🚫 Ingress Returns 404 on a Working Service

`Client requests /user` → `Ingress rule matches /api only` → `❌ 404`

**❓ The question:**
> You created an ingress, but all requests return 404 even though the service and pods are running. What could be wrong in the ingress configuration?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | `curl` the ingress endpoint | Request reaches the ingress, but gets a **404** — the failure is at the ingress layer, not the app |
| 2 | Inspect the ingress rule | Configured to forward traffic only when the path starts with **`/api`** |
| 3 | Compare to the actual request | Client is sending to **`/user`**, not `/api` — path mismatch, so the 404 is technically correct behavior |
| 4 | Update the ingress path to match the real client path | Request now matches the rule and routes to the backend service |

> 💡 **Takeaway:** An ingress 404 with a healthy service almost always means the **path rule** doesn't match what the client is actually requesting — check the literal path string first.

---

## 2. 🔓 TLS Configured but HTTP Still Works

```
  TLS secret attached  →  🔓 HTTPS enabled   but   HTTP still works  →  ❌ no redirect enforced
```

**❓ The question:**
> An ingress is configured with a TLS secret, but users can still access the application over HTTP. Why is HTTPS not being enforced?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Test the app over plain HTTP | It's still accessible — HTTPS isn't being enforced |
| 2 | Inspect the ingress config | TLS is configured via a secret, which **enables** HTTPS |
| 3 | Identify what's missing | TLS configuration only turns HTTPS *on* — it does **not** automatically redirect HTTP → HTTPS |
| 4 | Add an annotation to force HTTP → HTTPS redirection | E.g. `nginx.ingress.kubernetes.io/ssl-redirect: "true"` (or your controller's equivalent) |
| 5 | Verify | HTTP requests are now redirected to HTTPS — secure access enforced |

> 💡 **Takeaway:** Enabling TLS and *enforcing* TLS are two separate steps — the redirect annotation is what actually closes the HTTP door.

---

## 3. ✂️ Path Rewrite Not Taking Effect

**❓ The question:**
> A user sends a request to `/api/users` through an ingress configured with a path rewrite rule. The request reaches the backend, but the backend still receives `/api/users` instead of the expected `/v1/users`. Why isn't the rewrite taking effect?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Check backend logs | Backend receives `/api/users` — wrong, app expects `/v1/users` |
| 2 | Confirm what the client sends | Client correctly calls `/api/users` — client behavior isn't the issue |
| 3 | Inspect the ingress config | A rewrite annotation exists, but the path is defined as **`/api`** with **no capture group** — so the rewrite never actually triggers |
| 4 | Fix the ingress path | Use a path that **captures** everything after `/api` (e.g. `/api(/|$)(.*)`), rewriting the captured group to `/v1/$2` |
| 5 | Verify | Backend now receives `/v1/users` — rewrite applied correctly |

> 💡 **Takeaway:** A rewrite annotation does nothing without a **capture group** in the path regex — without one, there's nothing for the rewrite target to substitute.

---

## 4. 🎯 Exact Path Match Blocking Valid Requests

```
  Client requests:  /app
  Ingress rule:     /application   (pathType: Exact)   →   ❌ no match → 404
```

**❓ The question:**
> A user tries to access an application through an ingress using `example.com/app`. The service and pods are running and healthy, but every request returns 404. What's wrong in the ingress configuration?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe the failure | Immediate 404 — request doesn't match any ingress rule |
| 2 | Inspect the ingress YAML | Only matches the **exact** path `/application`, with `pathType: Exact` |
| 3 | Compare paths | Client requests `/app`, ingress is configured for `/application` — they don't match exactly |
| 4 | Update the path to `/app` and change `pathType` to **`Prefix`** | Any request starting with `/app` now routes correctly |
| 5 | Verify | Request matches the rule, routed successfully |

> 💡 **Takeaway:** `pathType: Exact` means *exact* — even a sensible-looking shorter path won't match. Use `Prefix` unless you specifically need exact matching.

---

## 5. 🔀 Wrong Service Receiving the Traffic

**❓ The question:**
> A user accesses `admin.example.com`, but instead of reaching the admin application, the request is served by the main application service. Both services are running, and a single ingress resource is used. What mistake in the ingress rules could cause this?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe the problem | Request succeeds, but the response belongs to the **main app**, not the admin app — traffic is routed, just to the wrong place |
| 2 | Inspect the ingress rules | The rule for `example.com` appears **before** the rule for `admin.example.com`, and both use a catch-all path |
| 3 | Identify the cause | Because `example.com` is a broader match, requests for `admin.example.com` get caught by it first |
| 4 | Reorder the ingress rules — more specific host rules before broader ones | Ensures correct routing precedence |
| 5 | Verify | Response now correctly comes from the admin application |

> 💡 **Takeaway:** Ingress rule order matters — a broad catch-all rule placed earlier can silently swallow traffic meant for a more specific host.

---

## 6. 📋 ConfigMap Exists, but App Crashes on Missing Config

```
  ConfigMap ✅ created, correct keys   →   Deployment ❌ never references it   →   💥 crash on startup
```

**❓ The question:**
> You created a ConfigMap to store application configuration and applied it successfully. After deploying the application, it crashes immediately on startup saying a required configuration value is missing. What's the root cause?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe the application failure | Crashes due to a missing config value at runtime — points to configuration injection, not app code |
| 2 | Verify the ConfigMap | Exists, contains the correct keys — ConfigMap creation isn't the problem |
| 3 | Inspect the deployment spec | **Doesn't reference the ConfigMap at all** — this is the root cause |
| 4 | Add `envFrom: configMapRef:` (or equivalent) to the container spec | Tells Kubernetes to inject all keys from the ConfigMap as environment variables |
| 5 | Verify | Application receives the required config and starts successfully |

> 💡 **Takeaway:** Creating a ConfigMap doesn't automatically wire it to anything — a pod only gets its values if the deployment **explicitly references it**.

---

## 7. 🌫️ envFrom Injects Empty Environment Variables

**❓ The question:**
> A deployment references a ConfigMap using `envFrom` to inject environment variables. The pod starts successfully, but inside the container, the expected environment variables are empty or missing — no errors shown in pod events. What could be wrong?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe the pod | Running fine, but expected env vars are missing — points to configuration injection, not startup/scheduling |
| 2 | Verify the ConfigMap | Exists with the correct keys — not a data issue |
| 3 | Inspect the deployment | Correctly references the ConfigMap — injection mechanism is configured |
| 4 | Check namespace alignment | Deployment runs in **`prod`**, but the ConfigMap exists in **`default`** — ConfigMaps are namespace-scoped and can't be shared across namespaces |
| 5 | Create the ConfigMap in the same namespace as the deployment, restart the pods | Environment variables are now injected correctly |

> 💡 **Takeaway:** ConfigMaps (and Secrets) are strictly namespace-scoped — a perfectly correct ConfigMap in the wrong namespace is functionally invisible to your pod.

---

## 8. 📁 Mounted Config File Turns Into a Directory

```
  Expected:  /etc/config/configmap.yaml   (a file)
  Actual:    /etc/config/configmap.yaml/  (a directory!) → app can't find the file
```

**❓ The question:**
> You mount a ConfigMap as a volume to provide a configuration file to an application. The pod is running, but the application fails saying the config file doesn't exist at the expected path. What common mistake causes this?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe the failure | App can't find `configmap.yaml` at the expected location |
| 2 | Verify the ConfigMap contains the file | It does — not a data/creation issue |
| 3 | Inspect the pod's mounted files | Kubernetes created a **directory** named `configmap.yaml`, with the actual file placed *inside* it — so the real path became `configmap.yaml/configmap.yaml` |
| 4 | Add `subPath: configmap.yaml` to the volume mount | Tells Kubernetes to mount only that single file at the expected path, instead of the whole ConfigMap as a directory |
| 5 | Verify | File now exists at the correct path, application starts successfully |

> 💡 **Takeaway:** Mounting a ConfigMap volume without `subPath` mounts it as a **directory** of all its keys — use `subPath` when you need a single file at a specific path.

---

## 9. 🔑 Secret Injected, but Credentials Are Stale

**❓ The question:**
> A Secret is created and used as an environment variable in a deployment. The pod starts, but the application fails database authentication — environment variables inside the container show unexpected values. What could have caused this?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe the failure | App runs but can't authenticate — pods/containers are healthy, but the **credentials are wrong** |
| 2 | Check env vars inside the container | Kubernetes is injecting *something*, but the password value doesn't match what the database currently expects — not an injection failure, but **wrong data being injected** |
| 3 | Inspect and decode the Kubernetes Secret | Kubernetes is correctly storing and passing the **old password** — the stored value itself is outdated |
| 4 | Identify the root cause | The DB password was changed **externally**, but the Kubernetes Secret was never updated — Kubernetes doesn't auto-sync with external systems |
| 5 | Update the Secret with the correct credential, restart the pod | New Secret values are injected, application authenticates successfully |

> 💡 **Takeaway:** Kubernetes Secrets are a **static snapshot**, not a live link to your external system of record — change the password there, and you must update the Secret too.

---

## 10. 🔄 ConfigMap Update Not Reflected in Running Pods

```
  Pod (2 days old):  feature_flag = old   ❌
  Pod (5 min old):   feature_flag = new   ✅
  → same ConfigMap, different values, because env vars load only at container start
```

**❓ The question:**
> You update a ConfigMap that contains a feature flag used by a running application. Some pods start using the new behavior while others continue using the old configuration — Kubernetes shows zero errors. Why?

**✅ The answer, step by step:**

| Step | Action | What It Tells You |
|------|--------|--------------------|
| 1 | Observe inconsistent behavior | Older pods (2 days old) behave differently from a newly created pod (5 minutes old) |
| 2 | Confirm the ConfigMap was updated | Yes — it has the new feature flag value |
| 3 | Compare env vars across pods | Old pods still hold the **old** value; the new pod has the **updated** value |
| 4 | Identify the root cause | ConfigMap is consumed as environment variables, which are **only read once, at container start** — running pods never see live updates |
| 5 | `kubectl rollout restart deployment/<name>` | Forces all pods to recreate and start fresh with the updated ConfigMap values |

> 💡 **Takeaway:** Environment-variable-based ConfigMap consumption is a one-time snapshot at startup — if you need live updates, mount the ConfigMap as a **volume** instead (which does update), or restart the deployment after every change.

---

```
   ✅ ────────────────────────────────────  ✅
              End of Set 4
   ✅ ────────────────────────────────────  ✅
```

<sub>Notes by **InfraCorps** — Set 4 of the Kubernetes Real-World Failures series. This completes the full 40-question series across all four sets.</sub>
