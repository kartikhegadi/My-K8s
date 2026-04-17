# 🚀 AWS EFS + EKS: Complete Deep-Dive Guide

> **A production-ready reference for setting up Amazon Elastic File System (EFS) with Amazon Elastic Kubernetes Service (EKS) — with architecture diagrams, real-world scenarios, and best practices.**

---

## 📑 Table of Contents

1. [What is This?](#what-is-this)
2. [Core Concepts Explained](#core-concepts-explained)
3. [Architecture Overview](#architecture-overview)
4. [Why Use EFS with EKS?](#why-use-efs-with-eks)
5. [Advantages & Disadvantages](#advantages--disadvantages)
6. [Real-World Scenario](#real-world-scenario)
7. [Step-by-Step Setup Guide](#step-by-step-setup-guide)
8. [How Data Flows](#how-data-flows)
9. [EFS vs Other Storage Options](#efs-vs-other-storage-options)
10. [Security Best Practices](#security-best-practices)
11. [Troubleshooting](#troubleshooting)
12. [Summary Cheatsheet](#summary-cheatsheet)

---

## 🧠 What is This?

This guide covers the complete integration of **Amazon EFS (Elastic File System)** with **Amazon EKS (Elastic Kubernetes Service)**. In plain terms:

| Component | What it is | Analogy |
|-----------|-----------|---------|
| **EKS** | Managed Kubernetes — runs your containerized apps | A smart factory that runs your apps |
| **EFS** | Managed NFS file storage — shared, scalable, persistent | A shared hard drive in the cloud |
| **EFS CSI Driver** | A Kubernetes plugin that connects EFS to your pods | A USB driver that lets your factory use the hard drive |
| **Mount Target** | An NFS endpoint inside each Availability Zone | The physical socket in each building of your factory |
| **Security Group** | A firewall that controls network access | A security guard checking who can enter |

---

## 🧩 Core Concepts Explained

### Amazon EFS (Elastic File System)
EFS is a **fully managed, serverless, elastic NFS (Network File System)** provided by AWS. Unlike a traditional hard drive, EFS:
- **Automatically grows and shrinks** as you add/remove files
- **Is accessible from multiple instances simultaneously** (ReadWriteMany)
- **Survives across Availability Zones** — highly available by design
- **Charges only for what you use** — no pre-provisioning needed

### Amazon EKS (Elastic Kubernetes Service)
EKS is AWS's **managed Kubernetes** service. Kubernetes (K8s) is an orchestration platform that:
- Runs your applications in containers (Docker)
- Automatically scales, restarts, and balances your workloads
- Uses **Pods** as the smallest unit (a pod = one or more containers)
- Uses **Persistent Volume Claims (PVCs)** to request storage

### Why Pods Need Persistent Storage
By default, **containers are ephemeral** — when a pod dies, its data is gone. For stateful workloads (databases, CMS, ML models, shared configs), you need storage that:
- Persists beyond pod restarts ✅
- Is accessible to multiple pods at once ✅
- Is managed outside Kubernetes ✅

**EFS fulfills all three.**

### CSI Driver (Container Storage Interface)
The **EFS CSI Driver** is a Kubernetes-native plugin that:
- Translates Kubernetes storage requests (PVC) → EFS API calls
- Mounts EFS volumes into pods transparently
- Is maintained by AWS and installed via Helm

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                          AWS Cloud                                    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                     VPC (Virtual Private Cloud)               │    │
│  │                                                                │    │
│  │   ┌─────────────────────┐    ┌─────────────────────────┐     │    │
│  │   │  Availability Zone A │    │  Availability Zone B     │     │    │
│  │   │                      │    │                           │     │    │
│  │   │  ┌───────────────┐   │    │  ┌───────────────────┐   │     │    │
│  │   │  │  EKS Node 1   │   │    │  │   EKS Node 2      │   │     │    │
│  │   │  │  ┌─────────┐  │   │    │  │  ┌─────────────┐  │   │     │    │
│  │   │  │  │  Pod A  │  │   │    │  │  │   Pod B     │  │   │     │    │
│  │   │  │  │  Pod B  │  │   │    │  │  │   Pod C     │  │   │     │    │
│  │   │  │  └────┬────┘  │   │    │  │  └──────┬──────┘  │   │     │    │
│  │   │  └───────┼───────┘   │    │  └─────────┼─────────┘   │     │    │
│  │   │          │            │    │             │              │     │    │
│  │   │  ┌───────▼──────────┐│    │  ┌──────────▼──────────┐  │     │    │
│  │   │  │  EFS Mount Target││    │  │  EFS Mount Target   │  │     │    │
│  │   │  │  (AZ-A endpoint) ││    │  │  (AZ-B endpoint)    │  │     │    │
│  │   │  └───────────────────┘│   │  └─────────────────────┘  │     │    │
│  │   └──────────────────────┘    └───────────────────────────┘     │    │
│  │                                                                    │    │
│  │                    ┌─────────────────────┐                        │    │
│  │                    │  Amazon EFS          │                        │    │
│  │                    │  (Shared Storage)    │                        │    │
│  │                    │  fs-xxxxxxxxxx        │                        │    │
│  │                    └─────────────────────┘                        │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

### Kubernetes Layer Detail

```
┌──────────────────────────────────────────────────────────────┐
│                    Kubernetes Layer                            │
│                                                                │
│  Pod  ──requests──►  PVC  ──bound to──►  PV  ──maps to──►  EFS│
│  (app)             (claim)           (volume)            (storage)│
│                                                                │
│  PVC = "I need 20Gi of ReadWriteMany storage"                 │
│  PV  = "Here is 20Gi backed by EFS fs-xxxxxxxx"               │
└──────────────────────────────────────────────────────────────┘
```

---

## ✅ Why Use EFS with EKS?

### The Core Problem EFS Solves

In Kubernetes, pods are **stateless and replaceable**. When you scale horizontally (multiple replicas), each pod runs on a **different node** in a **different AZ**. Traditional block storage (like EBS) is:
- **Tied to a single AZ** — can't be shared
- **Only mounted by one node at a time** — no `ReadWriteMany`
- **Not suitable for shared data** across replicas

EFS solves all of this:

```
Without EFS:                     With EFS:
                                  
Pod-1 → EBS-1 (AZ-A) ❌ shared  Pod-1 ──┐
Pod-2 → EBS-2 (AZ-B) ❌ shared  Pod-2 ──┼──► EFS (any AZ) ✅
Pod-3 → EBS-3 (AZ-C) ❌ shared  Pod-3 ──┘
Each pod has isolated storage!   All pods share one filesystem!
```

---

## ⚖️ Advantages & Disadvantages

### ✅ Advantages

| # | Advantage | Details |
|---|-----------|---------|
| 1 | **ReadWriteMany (RWX)** | Multiple pods across nodes/AZs can read and write simultaneously — impossible with EBS |
| 2 | **Fully Managed** | No OS patching, no capacity planning, no NFS server to maintain |
| 3 | **Elastic Scaling** | Auto-grows/shrinks from KB to petabytes; you pay only for what you use |
| 4 | **High Availability** | Data is stored across multiple AZs by default |
| 5 | **Durable** | 99.999999999% (11 nines) durability |
| 6 | **POSIX Compliant** | Standard filesystem semantics — apps that work with local files work with EFS |
| 7 | **Access Points** | Fine-grained access control per directory per application |
| 8 | **Lifecycle Management** | Automatically move infrequent files to cheaper IA storage tier |
| 9 | **Encryption** | At-rest and in-transit encryption built-in |
| 10 | **IAM Integration** | Can enforce AWS IAM policies at the file system level |

### ❌ Disadvantages

| # | Disadvantage | Details |
|---|--------------|---------|
| 1 | **Cost** | ~3× more expensive than EBS per GB; not ideal for high-volume block I/O |
| 2 | **Latency** | Higher latency than EBS (network-based NFS vs local block); not suitable for latency-sensitive DBs |
| 3 | **Throughput Limits** | Bursting mode has limits; need to provision throughput for sustained high I/O |
| 4 | **Not for Databases** | MySQL, PostgreSQL need block storage (EBS) — EFS NFS semantics aren't optimized for DB workloads |
| 5 | **Cold Start** | First access after long idle can be slower due to NFS caching behavior |
| 6 | **Complexity** | Requires Security Groups, Mount Targets per AZ, and CSI Driver installation — more setup than EBS |
| 7 | **No Windows Support** | NFS protocol — Linux-only workloads |
| 8 | **Network Dependency** | Performance depends on VPC network quality |

---

## 🏭 Real-World Scenario

### Scenario: Shared CMS (Content Management System) on EKS

**Company:** An e-commerce platform using WordPress or Drupal deployed on EKS with 3 replicas.

**Problem Without EFS:**
```
Replica-1 (AZ-A): User uploads product image → saved to /uploads on local pod storage
Replica-2 (AZ-B): Next request hits this pod → /uploads is EMPTY → image not found! 💥
Replica-3 (AZ-C): Same problem — each pod has its own isolated storage
```

**Solution With EFS:**
```
Replica-1, 2, 3 all mount the SAME EFS volume at /uploads
User uploads image via Replica-1 → saved to EFS
Next request hits Replica-2 → reads from same EFS → image found! ✅
```

**Real Architecture:**

```
┌──────────────────────────────────────────────────────────┐
│  WordPress EKS Deployment (3 replicas)                    │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │ WP Pod 1    │  │ WP Pod 2    │  │ WP Pod 3    │       │
│  │ (AZ-A)      │  │ (AZ-B)      │  │ (AZ-C)      │       │
│  │             │  │             │  │             │       │
│  │ /var/www/   │  │ /var/www/   │  │ /var/www/   │       │
│  │ html/wp-    │  │ html/wp-    │  │ html/wp-    │       │
│  │ content/    │  │ content/    │  │ content/    │       │
│  │ uploads/    │  │ uploads/    │  │ uploads/    │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                │                │               │
│         └────────────────┼────────────────┘               │
│                          │                                 │
│                    ┌─────▼──────┐                          │
│                    │  EFS PVC   │                          │
│                    │  (RWX)     │                          │
│                    └─────┬──────┘                          │
│                          │                                 │
│                    ┌─────▼──────┐                          │
│                    │ Amazon EFS │                          │
│                    │ fs-xxxxx   │                          │
│                    │            │                          │
│                    │ /uploads/  │                          │
│                    │   img1.jpg │                          │
│                    │   img2.png │                          │
│                    └────────────┘                          │
└──────────────────────────────────────────────────────────┘
```

**Other Real-World Use Cases:**

| Use Case | Why EFS |
|----------|---------|
| ML Model Serving | Multiple inference pods read the same model weights |
| CI/CD Build Cache | Build agents share cache across parallel jobs |
| Log Aggregation | Multiple app pods write logs to a shared directory |
| Config Management | All pods read the same config files |
| Media Processing | Upload pod writes, transcoding pods read the same file |
| Jupyter Notebooks | Data scientists share datasets across notebook pods |

---

## 📋 Step-by-Step Setup Guide

### Prerequisites Checklist

- [ ] AWS CLI installed and configured
- [ ] kubectl configured to connect to your EKS cluster
- [ ] Helm 3.x installed
- [ ] Worker node subnet IDs ready
- [ ] EKS worker node IAM role has EFS permissions

### Step 0: Check Pod Volume Assignments

Before anything, understand which pods use which volumes:

```bash
kubectl get pod -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
VOLUME:.spec.volumes[*].persistentVolumeClaim.claimName
```

**What you'll see:**
```
NAME           STATUS    VOLUME
webapp-pod-1   Running   webapp-pvc
webapp-pod-2   Running   webapp-pvc    ← Same PVC = shared storage
webapp-pod-3   Running   webapp-pvc
```

---

### Step 1: Get VPC ID

Your EKS nodes run inside a VPC. EFS must be created in the same VPC.

```bash
VPC_ID=$(aws ec2 describe-subnets \
  --subnet-ids <Node1-SubnetID> \
  --query "Subnets[0].VpcId" \
  --output text)

echo "VPC ID: $VPC_ID"
```

> 💡 **Tip:** Get your subnet IDs from:
> ```bash
> kubectl get nodes -o jsonpath='{.items[*].spec.providerID}'
> aws ec2 describe-instances --instance-ids <id> --query 'Reservations[0].Instances[0].SubnetId'
> ```

---

### Step 2: Create Security Group for EFS

A dedicated security group controls who can connect to EFS.

```bash
# Create the security group
EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Security group for EFS access from EKS" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)

echo "Security Group ID: $EFS_SG_ID"
```

#### Allow NFS Traffic (Port 2049)

```bash
# ⚠️  Dev/Testing: Open to all (NOT for production)
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --cidr 0.0.0.0/0

# ✅ Production: Restrict to EKS node security group only
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --source-group <EKS-Node-Security-Group-ID>
```

**Security Group Flow:**

```
EKS Nodes (sg-nodes) ──port 2049──► EFS SG (sg-efs) ──► EFS File System
        ↑
   Only this is allowed in production
```

---

### Step 3: Create EFS File System

```bash
EFS_ID=$(aws efs create-file-system \
  --creation-token kastro-efs-$(date +%s) \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=kastro-eks-efs \
  --query "FileSystemId" \
  --output text)

echo "EFS File System ID: $EFS_ID"
```

**Performance Mode Options:**

| Mode | Use Case | Max IOPS |
|------|----------|----------|
| `generalPurpose` | Web apps, CMS, CI/CD | 35,000 |
| `maxIO` | Big data, media processing, ML | Unlimited |

**Throughput Mode Options:**

| Mode | Use Case | Cost |
|------|----------|------|
| `bursting` | Variable workloads, smaller files | Pay per GB stored |
| `provisioned` | Sustained high throughput needed | Pay per MB/s provisioned |
| `elastic` | Unpredictable, spiky workloads | Pay per GB transferred |

---

### Step 4: Create Mount Targets (One Per AZ)

Mount targets are the **NFS endpoints** inside each AZ. Pods in AZ-A use the AZ-A mount target — this reduces latency and cross-AZ data transfer costs.

```bash
# Get subnet IDs for each worker node
NODE1_SUBNET="subnet-xxxxxxxx"   # AZ-A
NODE2_SUBNET="subnet-yyyyyyyy"   # AZ-B

# Mount Target for AZ-A
aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id $NODE1_SUBNET \
  --security-groups $EFS_SG_ID

# Mount Target for AZ-B
aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id $NODE2_SUBNET \
  --security-groups $EFS_SG_ID
```

**Mount Target Topology:**

```
         EFS File System (fs-xxxxxxxx)
                    │
       ┌────────────┼────────────┐
       │            │            │
 Mount Target  Mount Target  Mount Target
    AZ-A          AZ-B          AZ-C
  10.0.1.x     10.0.2.x      10.0.3.x
       │            │            │
  EKS Node 1  EKS Node 2  EKS Node 3
  (Pods here) (Pods here) (Pods here)
```

> ⚠️ **Critical:** Without a mount target in an AZ, pods on nodes in that AZ **cannot access EFS**. Always create one per AZ your nodes run in.

---

### Step 5: Verify Mount Targets

```bash
aws efs describe-mount-targets \
  --file-system-id $EFS_ID \
  --query "MountTargets[*].{AZ:AvailabilityZoneName,IP:IpAddress,State:LifeCycleState}"
```

**Expected Output:**
```json
[
  {
    "AZ": "ap-south-1a",
    "IP": "10.0.1.45",
    "State": "available"
  },
  {
    "AZ": "ap-south-1b",
    "IP": "10.0.2.67",
    "State": "available"
  }
]
```

Wait until `State` = `available` before proceeding.

---

### Step 6: Install EFS CSI Driver via Helm

The CSI driver is the bridge between Kubernetes and EFS.

```bash
# Add the Helm repository
helm repo add aws-efs-csi-driver \
  https://kubernetes-sigs.github.io/aws-efs-csi-driver/

helm repo update

# Install the driver
helm install aws-efs-csi-driver \
  aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set image.repository=602401143452.dkr.ecr.ap-south-1.amazonaws.com/eks/aws-efs-csi-driver
```

> 💡 Replace the ECR region (`ap-south-1`) with your region.

```bash
# Verify installation
kubectl get pods -n kube-system | grep efs-csi
```

**Expected Output:**
```
efs-csi-controller-xxxx   3/3   Running   0   2m
efs-csi-node-xxxxx        3/3   Running   0   2m   ← One per node
efs-csi-node-yyyyy        3/3   Running   0   2m
```

---

### Step 7: Create StorageClass

```yaml
# efs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap      # Access Point mode (recommended)
  fileSystemId: fs-xxxxxxxxxx   # Your EFS ID
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
reclaimPolicy: Retain            # Don't delete EFS data when PVC is deleted
volumeBindingMode: Immediate
```

```bash
kubectl apply -f efs-storageclass.yaml
```

---

### Step 8: Create PVC and Deploy Pod

```yaml
# efs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany      # ← Key difference from EBS (ReadWriteOnce)
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi       # EFS ignores this — it's elastic; just a placeholder
```

```yaml
# efs-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx
        volumeMounts:
        - name: persistent-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: persistent-storage
        persistentVolumeClaim:
          claimName: efs-claim
```

```bash
kubectl apply -f efs-pvc.yaml
kubectl apply -f efs-pod.yaml
kubectl get pvc   # Should show STATUS: Bound
kubectl get pods  # All 3 replicas Running
```

---

## 🔄 How Data Flows

```
User Request
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│  Step 1: Pod makes a file system call                    │
│  (e.g., open("/data/file.txt", "w"))                    │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Step 2: Linux kernel routes to NFS client               │
│  (EFS CSI driver mounted EFS via NFS v4.1 protocol)     │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Step 3: NFS request goes to EFS Mount Target           │
│  (the endpoint in the same AZ as the node)              │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Step 4: EFS stores data across multiple AZs            │
│  (data is replicated internally for durability)          │
└─────────────────────────────────────────────────────────┘

Complete path:
Pod → Node → Mount Target (same AZ) → EFS Backend (multi-AZ)
```

---

## 📊 EFS vs Other Storage Options

| Feature | EFS | EBS | S3 | FSx for Lustre |
|---------|-----|-----|----|----------------|
| **Type** | NFS | Block | Object | Parallel FS |
| **Access** | ReadWriteMany | ReadWriteOnce | HTTP API | ReadWriteMany |
| **Multi-AZ** | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes |
| **Protocol** | NFS v4 | iSCSI | REST | Lustre |
| **Latency** | ~1-3ms | ~0.1ms | ~50-100ms | <1ms |
| **Max Throughput** | Elastic | 16,000 IOPS | Unlimited | Terabytes/s |
| **Use Case** | Shared files | Databases | Backups/media | HPC, ML |
| **Price (per GB)** | ~$0.30 | ~$0.10 | ~$0.023 | ~$0.14 |
| **Kubernetes RWX** | ✅ Native | ❌ No | Via driver | ✅ Yes |

**When to choose what:**

```
Need shared storage across pods?     → EFS
Running a database (MySQL, Postgres)?→ EBS
Storing static assets, backups?      → S3
ML training, HPC workloads?          → FSx for Lustre
```

---

## 🔐 Security Best Practices

### 1. Restrict Security Group

```bash
# ❌ Bad: Open to all
--cidr 0.0.0.0/0

# ✅ Good: Only EKS nodes can connect
--source-group <EKS-Node-SG-ID>
```

### 2. Enable Encryption

```bash
# At-rest encryption (add to create-file-system)
--encrypted \
--kms-key-id arn:aws:kms:region:account:key/key-id

# In-transit: handled automatically by EFS CSI driver with TLS
```

### 3. Use EFS Access Points

Access Points create isolated directory views per application:

```yaml
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxxxxx
  directoryPerms: "700"
  uid: "1000"
  gid: "1000"
```

```
EFS Root (/)
├── /app1/     ← Access Point for App 1 (uid:1000)
├── /app2/     ← Access Point for App 2 (uid:2000)
└── /shared/   ← Shared between all
```

### 4. IAM Policies for EKS Nodes

Ensure your node IAM role has these permissions:

```json
{
  "Effect": "Allow",
  "Action": [
    "elasticfilesystem:DescribeAccessPoints",
    "elasticfilesystem:DescribeFileSystems",
    "elasticfilesystem:DescribeMountTargets",
    "elasticfilesystem:CreateAccessPoint",
    "elasticfilesystem:DeleteAccessPoint",
    "ec2:DescribeAvailabilityZones"
  ],
  "Resource": "*"
}
```

---

## 🔧 Troubleshooting

### Pod Stuck in `Pending` — PVC Not Bound

```bash
kubectl describe pvc efs-claim
kubectl describe pod <pod-name>
```

**Common Causes:**
| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `no nodes available` | CSI node pod not running | Check `kubectl get pods -n kube-system \| grep efs` |
| `AccessDenied` | IAM permissions missing | Attach EFS policy to node role |
| `timeout` | Security group misconfigured | Verify port 2049 is open |
| `mount.nfs: Connection refused` | No mount target in node's AZ | Create mount target in that AZ |

### Check CSI Driver Logs

```bash
# Controller logs (scheduling, provisioning)
kubectl logs -n kube-system deployment/efs-csi-controller -c csi-provisioner

# Node logs (actual mounting)
kubectl logs -n kube-system daemonset/efs-csi-node -c efs-plugin
```

### Test EFS Connectivity from a Pod

```bash
kubectl run test-pod --image=amazonlinux --restart=Never -- sleep 3600
kubectl exec -it test-pod -- bash

# Inside pod:
mount | grep nfs       # Should show EFS mount
df -h                  # Should show EFS filesystem
echo "test" > /data/test.txt   # Write test
cat /data/test.txt             # Read back
```

---

## 📝 Summary Cheatsheet

```
SETUP FLOW:
──────────────────────────────────────────────────────────────
1. Get VPC ID            → All resources must be in same VPC
2. Create Security Group → Allow port 2049 (NFS) from EKS nodes
3. Create EFS            → The actual shared filesystem
4. Create Mount Targets  → One per AZ (entry points for NFS)
5. Verify Mount Targets  → Wait for "available" state
6. Install CSI Driver    → Kubernetes ↔ EFS bridge (via Helm)
7. Create StorageClass   → Tell K8s how to provision EFS volumes
8. Create PVC            → Request storage (ReadWriteMany)
9. Deploy Pods           → Mount PVC; all pods share one EFS

KEY PORTS & PROTOCOLS:
──────────────────────────────────────────────────────────────
NFS v4.1:  Port 2049 (TCP)
TLS wrap:  Handled by EFS CSI driver automatically

KEY IDs TO SAVE:
──────────────────────────────────────────────────────────────
VPC_ID      = vpc-xxxxxxxxxxxxxxxxx
EFS_SG_ID   = sg-xxxxxxxxxxxxxxxxx
EFS_ID      = fs-xxxxxxxxxx

ACCESSMODES COMPARISON:
──────────────────────────────────────────────────────────────
ReadWriteOnce (RWO)  → EBS  → 1 pod on 1 node
ReadWriteMany (RWX)  → EFS  → Many pods on many nodes ✅
ReadOnlyMany  (ROX)  → EFS  → Many pods, read-only
```

---

## 📚 Further Reading

- [AWS EFS Documentation](https://docs.aws.amazon.com/efs/latest/ug/)
- [EFS CSI Driver GitHub](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
- [Amazon EKS Storage Best Practices](https://aws.github.io/aws-eks-best-practices/storage/)
- [EFS Performance Guide](https://docs.aws.amazon.com/efs/latest/ug/performance.html)

---

*Guide authored for production EKS deployments. Always test in a staging environment before applying to production.*
