# Simple StatefulSet Data Persistence Demo

## Why EBS CSI Driver is Required

Your EKS cluster comes with a default `gp2` storage class, but it uses the old in-tree EBS provisioner which is being deprecated. The EBS CSI driver is required because:

1. **Modern Standard**: AWS recommends using CSI drivers for all new deployments
2. **Better Features**: Supports encryption, snapshots, and volume expansion
3. **Future-Proof**: In-tree drivers are being removed in future Kubernetes versions
4. **Dynamic Provisioning**: Creates volumes automatically when pods need them

## Step 1: Install EBS CSI Driver (One Command)

```bash
# This installs the EBS CSI driver as an EKS add-on (simplest method)
aws eks create-addon \
  --cluster-name kastro-eks \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1

# Wait for it to be ready
aws eks wait addon-active \
  --cluster-name kastro-eks \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1
```

## Step 2: Attach IAM Role to Worker Nodes

```bash
# Role Name
"AmazonEBSCSIDriverPolicy"
```

## Step 3: Create Custom Storage Class

```bash
kubectl apply -f custom-storage-class.yaml
```

## Step 4: Deploy StatefulSet

```bash
kubectl apply -f nginx-statefulset.yaml
```

## Step 5: Test Data Persistence

```bash
# 1. Add data to pod-0
kubectl exec nginx-statefulset-0 -- sh -c "echo 'Hello from Pod 0 - $(date)' > /usr/share/nginx/html/data.txt"

# 2. Add data to pod-1  
kubectl exec nginx-statefulset-1 -- sh -c "echo 'Hello from Pod 1 - $(date)' > /usr/share/nginx/html/data.txt"

# 3. Verify each pod has its own data
kubectl exec nginx-statefulset-0 -- cat /usr/share/nginx/html/data.txt
kubectl exec nginx-statefulset-1 -- cat /usr/share/nginx/html/data.txt

# 4. Delete both pods
kubectl delete pod nginx-statefulset-0 nginx-statefulset-1

# 5. Wait for pods to recreate (StatefulSet will recreate them automatically)
kubectl get pods -w

# 6. Verify data persisted after pod recreation
kubectl exec nginx-statefulset-0 -- cat /usr/share/nginx/html/data.txt
kubectl exec nginx-statefulset-1 -- cat /usr/share/nginx/html/data.txt
```

## What Happens During This Demo

1. **StatefulSet Creates Pods**: Each pod gets a unique name (nginx-statefulset-0, nginx-statefulset-1)
2. **Persistent Volumes Created**: Each pod gets its own 1GB EBS volume automatically
3. **Data Written**: Each pod writes to its own persistent storage
4. **Pods Deleted**: You manually delete the pods
5. **Pods Recreated**: StatefulSet automatically recreates pods with same names
6. **Data Persists**: New pods mount the same volumes and data is still there

## Key Concepts Explained

### StatefulSet vs Deployment
- **Deployment**: Pods are interchangeable, no persistent identity
- **StatefulSet**: Pods have stable names and persistent storage

### Volume Claim Templates
- Creates one PVC per pod automatically
- Each pod gets its own dedicated storage
- Storage survives pod deletion and recreation

### Storage Class
- Defines what type of storage to create
- `gp3` is faster and cheaper than `gp2`
- `WaitForFirstConsumer` ensures volume is created in same AZ as pod

This demo proves that StatefulSet maintains data persistence across pod lifecycles!
