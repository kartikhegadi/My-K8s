# ☸️ Kubernetes — Day 03

---

## 📋 Tasks

| # | Topic | Link |
|---|-------|------|
| 1 | Ecommerce Application | [▶ Watch on YouTube](https://www.youtube.com/watch?v=l-5JQcI_CH0&list=PLs-PsDpuAuTfG3gFR5DnVD58kT7JBO97x&index=3) |
| 2 | Microservices Project | [▶ Watch on YouTube](https://www.youtube.com/watch?v=3fhvnsNY5fc&list=PLs-PsDpuAuTfG3gFR5DnVD58kT7JBO97x&index=4) |

---

## 🐤 Canary Deployment

### Origin of "Canary"

> Originally from **coal mining** — miners used canary birds because they are highly sensitive to toxic gases. If dangerous gas appeared, the canary got affected first → giving miners an **early warning** to escape.

In Kubernetes, the same idea applies: roll out a new version to a **small subset** of users first to catch problems early.

### Traffic Split Architecture

```
🔵 STABLE  (v1 — 90%)  ─────────────────\
                                          ▶  🌐 LOAD BALANCER / INGRESS  ▶  User Traffic
🟢 CANARY  (v2 — 10%)  ─────────────────/
```

### Gradual Traffic Shift Strategy

| Stage | v1 Pods | v2 Pods | v2 Traffic % |
|-------|---------|---------|--------------|
| Initial | 4 | 1 | ~20% |
| Mid | 3 | 2 | ~40% |
| Final | 0 | 5 | 100% |

---

### 📄 Manifest Files

#### 1️⃣ `deployment-v1.yaml` — Stable (Blue Page)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
spec:
  replicas: 4   # 100% traffic initially
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args:
          - "-c"
          - |
            echo '<html><body style="background-color:blue;">
            <h1 style="color:white;">THIS IS VERSION 1</h1>
            </body></html>' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'
```

#### 2️⃣ `deployment-v2.yaml` — Canary (Green Page)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
spec:
  replicas: 1   # Start with ~20% traffic
  selector:
    matchLabels:
      app: nginx
      version: v2
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args:
          - "-c"
          - |
            echo '<html><body style="background-color:green;">
            <h1 style="color:white;">THIS IS VERSION 2 (CANARY)</h1>
            </body></html>' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'
```

#### 3️⃣ `service-lb.yaml` — LoadBalancer Service (routes to both v1 + v2)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx   # matches BOTH v1 and v2 pods
  ports:
  - port: 80
    targetPort: 80
```

### ⚙️ Commands

```bash
# Deploy everything
kubectl apply -f deployment-v1.yaml
kubectl apply -f deployment-v2.yaml
kubectl apply -f service-lb.yaml

kubectl get svc

# Scale to shift traffic gradually
kubectl scale deployment nginx-v1 --replicas=2
kubectl scale deployment nginx-v2 --replicas=5

# Full cutover to v2
kubectl scale deployment nginx-v1 --replicas=0
kubectl scale deployment nginx-v2 --replicas=5
```

---

## 💾 Volumes

### ❓ Problem: Data is NOT Persistent by Default

```
Pod created → exec in → write file → delete pod → new pod created → ❌ file is GONE
```

#### Demo: `no-vol.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-no-volume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-no-volume-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30000
```

#### Reproducing the Data Loss

```bash
kubectl apply -f no-vol.yaml
kubectl exec -it <pod-name> -- /bin/sh

# Inside pod:
cd /usr/share/nginx/html
echo "welcome to k8s" > hello.html

# Access: http://<Pub-IP>:30000/hello.html  ✅

# Now delete the pod
kubectl delete -f no-vol.yaml
kubectl apply -f no-vol.yaml
kubectl exec -it <new-pod-name> -- /bin/sh

# Inside new pod:
ls /usr/share/nginx/html
# Output: 50x.html  index.html   ← hello.html is GONE ❌
```

> **Conclusion:** Data written inside a pod is **not persistent**. We need Persistent Volumes.

---

## 🗄️ Kubernetes Volumes — PV & PVC

### Key Concepts

| Term | Full Form | Description |
|------|-----------|-------------|
| **PV** | Persistent Volume | Actual storage resource in the cluster |
| **PVC** | Persistent Volume Claim | Request for storage by a pod |
| **SC** | Storage Class | Defines how a volume is dynamically created |

### Storage Provisioning Types

| Type | Description |
|------|-------------|
| **Static Provisioning** | Admin manually creates PV |
| **Dynamic Provisioning** ⭐ | Kubernetes auto-creates PV via StorageClass |

### Architecture

```
+----------+        +----------+        +-------+
|    PV    | ──────▶|   PVC    | ──────▶|  POD  |
|  10 GB   |        |  10 GB   |        |       |
+----------+        +----------+        +-------+
     ▲
     │  (created by)
+----------+
|  AWS EBS |
+----------+
```

### Reclaim Policies

| Policy | Behavior |
|--------|----------|
| `Retain` | Volume stays after PVC is deleted (manual cleanup required) |
| `Delete` | Volume is automatically deleted with PVC |
| `Recycle` | ⚠️ Deprecated |

### Access Modes

| Mode | Short | Description |
|------|-------|-------------|
| ReadWriteOnce | `RWO` | One pod on one node can read & write |
| ReadOnlyMany | `ROX` | Multiple pods can read, none can write |
| ReadWriteMany | `RWX` | Multiple pods can read and write |

> 💡 **AWS Note:** EBS supports only `RWO` · EFS supports `RWX`

---

## 🔧 Setting Up EBS CSI Driver (Dynamic Provisioning on AWS)

### Pre-requisites

1. **IAM:** Attach `AmazonEBSCSIDriverPolicy` to your worker node IAM role
2. **OIDC Provider:**
   ```bash
   eksctl utils associate-iam-oidc-provider \
     --cluster kartik-cluster \
     --region ap-south-1 \
     --approve
   ```
3. **Check if EBS CSI is already installed:**
   ```bash
   kubectl get pods -n kube-system | grep ebs-csi
   ```

### Install EBS CSI Driver via Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Add the EBS CSI Helm repo
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm repo list

# Install the driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
    --namespace kube-system \
    --create-namespace \
    --set image.repository=602401143452.dkr.ecr.ap-south-1.amazonaws.com/eks/aws-ebs-csi-driver

# Verify
kubectl get pods -n kube-system | grep ebs-csi
```

**Expected output:**
```
ebs-csi-controller-67dc977479-fv9fr   5/5   Running   0   102s
ebs-csi-controller-67dc977479-vpj5t   5/5   Running   0   102s
ebs-csi-node-lls7s                    3/3   Running   0   99s
ebs-csi-node-mwf45                    3/3   Running   0   102s
```

---

## 📦 Persistent Volume — Practical (Dynamic EBS)

### Files Needed: `sc.yaml` · `pvc.yaml` · `deployment.yaml` · `servicemanifestfile.yaml`

#### 1️⃣ `sc.yaml` — StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc           # Referenced by PVC
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer   # Delay until pod is scheduled
reclaimPolicy: Retain
# allowVolumeExpansion: true             # Uncomment to allow resize

parameters:
  type: gp3              # General Purpose SSD
```

#### 2️⃣ `pvc.yaml` — PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ebs-sc   # Must match sc.yaml
```

#### 3️⃣ `deployment.yaml` — Nginx with EBS Volume

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ebs
  template:
    metadata:
      labels:
        app: nginx-ebs
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: ebs-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: ebs-storage
          persistentVolumeClaim:
            claimName: ebs-pvc
```

#### 4️⃣ `servicemanifestfile.yaml` — LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ebs-service
spec:
  type: LoadBalancer
  selector:
    app: nginx-ebs
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### ⚙️ Commands & Verification

```bash
# Deploy all resources
kubectl apply -f sc.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
kubectl apply -f servicemanifestfile.yaml

kubectl get pods
kubectl get pvc

# Exec into pod
kubectl exec -it nginx-deployment-<id> -- /bin/bash
cd /usr/share/nginx/html
# You'll see: lost+found  ← EBS volume mounted (initially empty)

# Write test files
echo "hello from ebs volume" > index.html
echo "hello from ebs kartik" > index1.html
# Access via LoadBalancer URL ✅

# Delete pod — Kubernetes recreates it
kubectl delete pod nginx-deployment-<id>

# Exec into new pod
kubectl exec -it nginx-deployment-<new-id> -- /bin/bash
ls /usr/share/nginx/html
# Output: index.html  index1.html  lost+found  ← Data is PERSISTENT ✅
```

### Cleanup

```bash
# Delete manifests (EBS volume stays because reclaimPolicy: Retain)
kubectl delete -f sc.yaml -f pvc.yaml -f deployment.yaml -f servicemanifestfile.yaml

# ⚠️ Manually delete the EBS volume from AWS Console if needed
```

---

## 🗃️ StatefulSet

Used when pods need:
- **Stable identity** (e.g., `nginx-0`, `nginx-1`, `nginx-2` — names don't change)
- **Persistent storage per pod**
- Mainly for **database workloads**

### ❌ Problem with Deployments for Databases

```bash
# Demo: MySQL with emptyDir (no persistence)
kubectl apply -f deployment-mysql.yaml
```

**`deployment-mysql.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        - name: MYSQL_DATABASE
          value: "testdb"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        emptyDir: {}   # ⚠️ Data is lost when pod is deleted!
```

#### Reproduce Data Loss

```bash
# Get pod name
kubectl get pods

# Create DB + insert data
kubectl exec -it mysql-deployment-<id> -- mysql -uroot -ppassword -e "CREATE DATABASE testdb;"
kubectl exec -it mysql-deployment-<id> -- mysql -uroot -ppassword -D testdb -e "CREATE TABLE test_table (id INT);"
kubectl exec -it mysql-deployment-<id> -- mysql -uroot -ppassword -D testdb -e "INSERT INTO test_table VALUES (1), (2), (3);"
kubectl exec -it mysql-deployment-<id> -- mysql -uroot -ppassword -D testdb -e "SELECT * FROM test_table;"

# Delete pod (simulates failure)
kubectl delete pod mysql-deployment-<id>

# Wait for new pod
kubectl get pods -w

# Check data in new pod — IT'S GONE ❌
kubectl exec -it $(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -uroot -ppassword -D testdb -e "SELECT * FROM test_table;"

# Error: Table 'testdb.test_table' doesn't exist
```

> **Fix:** Use **StatefulSet + PVC** to give each pod its own persistent volume.

---

## 👾 DaemonSet

> Use **DaemonSet** when you want **exactly one pod per node** for a specific workload.

Common use cases: log collectors, monitoring agents, node-level utilities.

```
Node 1  →  [DaemonSet Pod]
Node 2  →  [DaemonSet Pod]
Node 3  →  [DaemonSet Pod]
```

---

## 📌 Quick Reference

```bash
# Canary scaling
kubectl scale deployment <name> --replicas=<n>

# Storage
kubectl get sc          # List StorageClasses
kubectl get pv          # List PersistentVolumes
kubectl get pvc         # List PersistentVolumeClaims

# Exec into pod
kubectl exec -it <pod-name> -- /bin/bash

# Watch pods
kubectl get pods -w
```
