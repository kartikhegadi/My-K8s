# Kubernetes Ingress, Autoscaling & ArgoCD Notes

## Ingress Controller

An Ingress Controller is a reverse proxy/load balancer that routes external HTTP/HTTPS traffic to the correct Kubernetes services based on rules defined in the Ingress.

| Page          | Port |
|---------------|------|
| Home Page     | 81   |
| Products Page | 82   |
| Cart Page     | 83   |

- **Port Based Routing** — `PublicIP:82`
- **Path Based Routing** — routes based on URL path (e.g. `/app1`, `/app2`)

### Ingress Controller Demo (Architecture)

```
                                  USER
+---------------------------+    |       +----------------------------------+
|    Developer Inputs       |    |      |        Ingress Config            |
|---------------------------|    |      |----------------------------------|
| Dockerfile1,2,3           |    |      | ingress.yml                      |
| Image1, Image2, Image3    |    |      | microservices configuration      |
| Push images -> Dockerhub  |    |      | rules configuration              |
| YML: image1:v1            |    |      | Nginx, AWS LB, Kong,             |
| Ingress yml file          |    |      | HAProxy, Traefik                 |
+---------------------------+    |      +----------------------------------+

                                 |
                               \ | /
+---------------------------------------------------------------+
|                     Ingress Controller                        |
|                  /app1     /app2     /app3                    |
+---------------------------------------------------------------+
        |                    |                     |
   (LB URL 1)           (LB URL 2)            (LB URL 3)
        |                    |                     |
        v                    v                     v
+----------------+  +----------------+  +----------------+
| Microservice 1 |  | Microservice 2 |  | Microservice 3 |
|    /app1       |  |    /app2       |  |    /app3       |
+----------------+  +----------------+  +----------------+
service type:       service type:       service type:
ClusterIP           ClusterIP           ClusterIP

                                  +----------------------------+     NGINX
                                  | service type: ClusterIP    |     AWS LB
                                  | 3 deployment manifest files|     Kong
                                  | 3 service manifest files   |     HAProxy
                                  +----------------------------+     Traefik
```

---

## Installation

```bash
# Install Docker
apt install docker.io

# Install tree
apt install tree -y
```

### Install Ingress Controller

To install ingress, first install the nginx ingress controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```

```bash
kubectl get ing   # shows ingress service, no ingress service yet
```

**Verify the ingress pods:**

```bash
kubectl get pods -n ingress-nginx
```

**Verify the ingress service:**

```bash
kubectl get svc -n ingress-nginx   # You will see the load balancer URL
```

Example load balancer URL:
```
aaf62657d0dc04b35be250af4d6e47f7-1092281608.ap-south-1.elb.amazonaws.com
```

We will use the above URL to access the app.

---

## Practicals

### Directory Structure

```
.
├── app1
│   ├── Dockerfile
│   └── index.html
├── app2
│   ├── Dockerfile
│   └── index.html
├── app3
│   ├── Dockerfile
│   └── index.html
└── k8s
    ├── app1-deploy.yaml
    ├── app2-deploy.yaml
    ├── app3-deploy.yaml
    └── ingress.yaml
```

### CreateFiles.sh

```bash
#!/bin/bash

# Create main directories
mkdir -p app1 app2 app3 k8s

# Create files inside app folders
touch app1/Dockerfile app1/index.html
touch app2/Dockerfile app2/index.html
touch app3/Dockerfile app3/index.html

# Create Kubernetes YAML files
touch k8s/app1-deploy.yaml
touch k8s/app2-deploy.yaml
touch k8s/app3-deploy.yaml
touch k8s/ingress.yaml

echo "✅ Project structure created successfully!"

# Optional: show structure
tree .
```

### Dockerfile

Same Dockerfile is used in all three app directories:

```dockerfile
FROM nginx:alpine
COPY ./index.html /usr/share/nginx/html/index.html
```

### HTML Files

#### `app1/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>App 1 - Ingress Demo</title>
  <style>
    body {
      margin: 0;
      background: linear-gradient(135deg, #ff4e50, #f9d423);
      font-family: 'Segoe UI', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      color: #fff;
      text-align: center;
      animation: fadeIn 1s ease-in;
    }
    h1 {
      font-size: 4rem;
      margin-bottom: 10px;
      text-shadow: 2px 2px 8px rgba(0,0,0,0.3);
    }
    p {
      font-size: 1.5rem;
      max-width: 600px;
      line-height: 1.5;
    }
    @keyframes fadeIn {
      from { opacity: 0; transform: translateY(-10px); }
      to { opacity: 1; transform: translateY(0); }
    }
  </style>
</head>
<body>
  <div>
    <h1>🚀 App 1 Loaded</h1>
    <p>This is the futuristic red version of App 1, powered by Kubernetes & Ingress on AWS EKS.</p>
  </div>
</body>
</html>
```

#### `app2/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>App 2 - Ingress Demo</title>
  <style>
    body {
      margin: 0;
      background: linear-gradient(135deg, #56ab2f, #a8e063);
      font-family: 'Helvetica Neue', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      color: #fff;
      text-align: center;
      animation: slideUp 1s ease-in-out;
    }
    h1 {
      font-size: 3.5rem;
      margin-bottom: 15px;
    }
    p {
      font-size: 1.4rem;
    }
    @keyframes slideUp {
      from { opacity: 0; transform: translateY(20px); }
      to { opacity: 1; transform: translateY(0); }
    }
  </style>
</head>
<body>
  <div>
    <h1>🌿 App 2 is Live</h1>
    <p>This is App 2, deployed in Kubernetes and beautifully routed via Ingress.</p>
  </div>
</body>
</html>
```

#### `app3/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>App 3 - Ingress Demo</title>
  <style>
    body {
      margin: 0;
      background: linear-gradient(135deg, #1e3c72, #2a5298);
      font-family: 'Roboto', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      color: #ffffffcc;
      text-align: center;
      animation: zoomIn 1s ease;
    }
    h1 {
      font-size: 4rem;
      margin-bottom: 10px;
    }
    p {
      font-size: 1.3rem;
    }
    @keyframes zoomIn {
      from { opacity: 0; transform: scale(0.95); }
      to { opacity: 1; transform: scale(1); }
    }
  </style>
</head>
<body>
  <div>
    <h1>🌊 Welcome to App 3</h1>
    <p>This is App 3 — clean, responsive, and routed with Kubernetes Ingress on EKS.</p>
  </div>
</body>
</html>
```

---

## Manifest Files (k8s folder)

### `app1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - name: app1
          image: kartik404/app1-image:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service  # this service name will be referenced in the ingress yml file
spec:
  selector:
    app: app1
  ports:
    - port: 80
      targetPort: 80
```

### `app2.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
        - name: app2
          image: kartik404/app2-image:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service  # this service name will be referenced in the ingress yml file
spec:
  selector:
    app: app2
  ports:
    - port: 80
      targetPort: 80
```

### `app3.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
  template:
    metadata:
      labels:
        app: app3
    spec:
      containers:
        - name: app3
          image: kartik404/app3-image:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app3-service  # this service name will be referenced in the ingress yml file
spec:
  selector:
    app: app3
  ports:
    - port: 80
      targetPort: 80
```

### `ingress.yml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-service
                port:
                  number: 80
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2-service
                port:
                  number: 80
          - path: /app3
            pathType: Prefix
            backend:
              service:
                name: app3-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-service
                port:
                  number: 80
```

---

## Build Docker Images

```bash
docker login -u kartik404
# passwd: ********

docker build -t app1-image /app1
docker build -t app2-image /app2
docker build -t app3-image /app3

docker tag app1-image kartik404/app1-image:latest
docker tag app2-image kartik404/app2-image:latest
docker tag app3-image kartik404/app3-image:latest

docker push kartik404/app1-image:latest
docker push kartik404/app2-image:latest
docker push kartik404/app3-image:latest

docker images

kubectl apply -f ./k8s
```

### Useful Ingress Commands

```bash
kubectl get ingress
kubectl get ing

kubectl describe ingress apps-ingress
kubectl get svc -n ingress-nginx
kubectl get pods -A
kubectl get pods
kubectl get svc

kubectl get deploy
```

---

## 2048 Game with SSL Certificate

### Configure `kubectl` for EKS

```bash
aws eks update-kubeconfig --region <ClusterRegion> --name <ClusterName>
```

Example:
```bash
aws eks update-kubeconfig --region ap-south-1 --name kartik-cluster
```

Set your cluster name and region in environment variables:

```bash
cluster_name=<ClusterName>
export AWS_REGION=<ClusterRegion>
```

Example:
```bash
cluster_name=kartik-cluster
export AWS_REGION=ap-south-1
```

### OIDC Provider Setup

Extract the OIDC ID from your cluster:

```bash
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Check if an IAM OIDC provider already exists for your cluster:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```
> If output is returned: you already have an IAM OIDC provider and can skip the next step.

If no output is returned, create an IAM OIDC provider for your cluster:

```bash
eksctl utils associate-iam-oidc-provider --cluster <ClusterName> --approve --region <ClusterRegion>
```

Example:
```bash
eksctl utils associate-iam-oidc-provider --cluster kartik-cluster --approve --region ap-south-1
```

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

You will see the output — the same OIDC provider will also be visible in the EKS Console:
> EKS → Open the Cluster → Under "Overview" you will see "OpenID Connect provider URL" → You can verify the OIDC there.

### Configure AWS Load Balancer Controller

Download the IAM policy required for the AWS Load Balancer Controller:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
ls
```

You will see a file called `iam_policy.json`.

Create the IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

The name of the policy created is `AWSLoadBalancerControllerIAMPolicy`. The same policy will appear in the IAM console — go to the IAM console and verify, then copy the ARN of the policy created.

Example ARN:
```
arn:aws:iam::037151145720:policy/AWSLoadBalancerControllerIAMPolicy
```

#### Create the IAM Service Account

Create the IAM service account for the AWS Load Balancer Controller using `eksctl`. This command attaches the policy to the service account and (if it exists) overrides the existing service account:

```bash
eksctl create iamserviceaccount \
    --cluster=<ClusterName> \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=<ARNofThePolicy> \
    --override-existing-serviceaccounts \
    --region <ClusterRegion> \
    --approve
```

Example:
```bash
eksctl create iamserviceaccount \
    --cluster=kartik-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::037151145720:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-south-1 \
    --approve
```

Verify:

```bash
kubectl get sa -n kube-system
```
> You will see `aws-load-balancer-controller` got created.

#### Install the AWS Load Balancer Controller using Helm

```bash
# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Add the Helm repository and update it:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the AWS Load Balancer Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<ClusterName> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Example:
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=kartik-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### Verify the Controller Deployment

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Deploy the 2048 Game Application

The following YAML files create a dedicated namespace, deploy the 2048 game, create a service, and set up an ingress to expose the application via the ALB.

#### Namespace (`namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: 2048-game
```

Apply the namespace:

```bash
kubectl apply -f namespace.yaml
```

#### Deployment (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-2048-deployment
  namespace: 2048-game
  labels:
    app: game-2048
spec:
  replicas: 2
  selector:
    matchLabels:
      app: game-2048
  template:
    metadata:
      labels:
        app: game-2048
    spec:
      containers:
      - name: game-2048
        image: thipparthiavinash/2048-game
        ports:
        - containerPort: 80
```

Deploy the application:

```bash
kubectl apply -f deployment.yaml
```

#### Service (`service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-2048-service
  namespace: 2048-game
  labels:
    app: game-2048
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: game-2048
```

Create the service:

```bash
kubectl apply -f service.yaml
```

#### Ingress (`ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-2048-ingress
  namespace: 2048-game
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: game-2048-service
            port:
              number: 80
```

Create the ingress:

```bash
kubectl apply -f ingress.yaml
```

#### Updated Ingress with HTTPS (`ingress-https.yml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-2048-ingress
  namespace: 2048-game
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <Your-Cert-ARN-From-ACM-Console>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
spec:
  rules:
  - host: game.learnaws.today
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: game-2048-service
            port:
              number: 80
```

Create the ingress:

```bash
kubectl apply -f ingress-https.yml
```

---

## Autoscaling in Kubernetes

- **Horizontal Pod Autoscaling (HPA)**
- **Vertical Pod Autoscaling (VPA)**

### Types of Autoscaling

- Memory based scaling
- CPU based scaling

```
CPU > 80%  ---> Increase the pod count
CPU < 10%  ---> Decrease the pod count
```

> Install the Metrics Server to perform autoscaling. If you are working with cloud-managed clusters, the metrics server is usually installed by default.

### Autoscaling Diagram

```
================================================================================
           KUBERNETES AUTOSCALING — SYSTEMATIC DIAGRAM
================================================================================


  ╔══════════════════════════════════════════════════════════════════════════╗
  ║         ↔  HORIZONTAL POD AUTOSCALING (HPA)                             ║
  ║             "Scale OUT — add more pods"                                  ║
  ╚══════════════════════════════════════════════════════════════════════════╝

   HIGH TRAFFIC
   🌐🌐🌐🌐🌐
       │
       ▼
  ┌─────────────┐    reports     ┌──────────────┐    scales    ┌─────────────┐
  │ 📊 Metrics  │ ────────────► │  🤖 HPA      │ ──────────► │ 🗂️ Deploy-  │
  │   Server    │               │  Controller  │              │   ment      │
  └─────────────┘               └──────────────┘              └──────┬──────┘
                                                                      │
                                         ┌────────────────────────────┘
                                         │  creates replicas
                                         ▼
          BEFORE                                      AFTER
          ───────                                     ──────
        ┌────────┐                ┌────────┐  ┌────────┐  ┌────────┐
        │📦 Pod 1│                │📦 Pod 1│  │📦 Pod 2│  │📦 Pod 3│
        │        │    ────────►   │        │  │  NEW   │  │  NEW   │
        │cpu: 1  │                │cpu: 1  │  │cpu: 1  │  │cpu: 1  │
        │ram: 1G │                │ram: 1G │  │ram: 1G │  │ram: 1G │
        └────────┘                └────────┘  └────────┘  └────────┘
         1 pod                    ◄──────────────────────────────────►
                                              3 pods (scaled OUT)

  KEY POINTS:
  • Each pod keeps the SAME resource limits (cpu, ram unchanged)
  • Total capacity increases by adding more identical pods
  • 🔼 Load increases  →  HPA adds pods (scale out)
  • 🔽 Load decreases  →  HPA removes pods (scale in)


================================================================================


  ╔══════════════════════════════════════════════════════════════════════════╗
  ║         ↕  VERTICAL POD AUTOSCALING (VPA)                               ║
  ║             "Scale UP — make the pod bigger"                             ║
  ╚══════════════════════════════════════════════════════════════════════════╝

   HIGH TRAFFIC
   🌐🌐🌐🌐🌐
       │
       ▼
  ┌─────────────┐    reports     ┌──────────────┐
  │ 📊 Metrics  │ ────────────► │  🤖 VPA      │
  │   Server    │               │  Controller  │
  └─────────────┘               └──────┬───────┘
                                        │
                                        │  restart & resize pod
                                        ▼
          BEFORE                                      AFTER
          ───────                                     ──────
        ┌────────┐                                  ┌────────┐  ▲
        │📦 Pod 1│                                  │📦 Pod 1│  │
        │        │                                  │        │  │ BIGGER
        │cpu: 0.5│    ────────►                     │cpu: 2  │  │ (scaled
        │ram:256M│                                  │ram: 1G │  │  UP)
        └────────┘                                  └────────┘  ▼

         1 pod                                       1 pod
         small                                       LARGER ✨

  KEY POINTS:
  • Number of pods stays the SAME (no new pods added)
  • Pod is restarted with higher CPU and RAM limits
  • 🔼 Load increases  →  VPA resizes pod (scale up)
  • 🔽 Load decreases  →  VPA shrinks pod (scale down)


================================================================================
                          SIDE-BY-SIDE COMPARISON
================================================================================

                    HPA ↔                          VPA ↕
              ─────────────────────          ─────────────────────
              Pod count  CHANGES             Pod count  SAME
              Pod size   SAME                Pod size   CHANGES
              Strategy   Scale OUT           Strategy   Scale UP
              Risk       Slower start        Risk       Pod restart
              Best for   Stateless apps      Best for   Stateful apps
              ─────────────────────          ─────────────────────

                     ⚡ COMBINE BOTH for maximum efficiency ⚡

================================================================================
```

### Horizontal Pod Autoscaler (HPA)

#### Deployment & Service (`cpu-php-apache.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

#### CPU-Based Scaling (`cpu-hpa.yml`)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10
```

Generate load to trigger CPU scaling:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

#### Deployment & Service for Memory Scaling (`mem-php-apache.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 100Mi   # Guaranteed memory per pod
          limits:
            cpu: 500m
            memory: 200Mi   # Max memory pod can use. If exceeded, pod is OOMKilled
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  type: ClusterIP
  selector:
    run: php-apache
  ports:
  - port: 80
    targetPort: 80
```

#### Memory-Based Scaling (`mem-hpa.yml`)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-memory
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 10Mi
```

> If average memory usage per pod exceeds 10Mi, HPA starts scaling.

Generate load:

```bash
kubectl run -i --tty load-generator \
  --rm \
  --restart=Never \
  --image=busybox:1.28 \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

---

## ArgoCD

Any deployments done in Kubernetes should be automated using a GitOps approach with ArgoCD.

- **Jenkins** — CI (clone the code, package the code, build the image, tag the image, push the image)
- **ArgoCD** — CD (monitors changes in the K8s manifest files; ArgoCD understands the changes in the YAMLs and then deploys the app accordingly)

### Setup ArgoCD using Helm

#### Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

#### Install ArgoCD using Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
```

Install ArgoCD in the `argocd` namespace:

```bash
helm install argocd argo/argo-cd --namespace argocd
```

```bash
kubectl get all -n argocd
```

> You will see multiple resources running. Under "services" you can see `argo-cd server` with type `ClusterIP`. To access it outside the cluster, we need a Load Balancer, so we'll patch this ClusterIP to LoadBalancer.

#### Expose ArgoCD Server

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash
kubectl get all -n argocd
```

> Now you can see the `argo-cd server` service changed from ClusterIP to LoadBalancer. Copy the load balancer URL.

```bash
yum install jq -y
```

```bash
kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'
```

#### Get ArgoCD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## Task

Reference video: [https://youtu.be/3fhvnsNY5fc?si=U6gX5T9xMQrfMt_P](https://youtu.be/3fhvnsNY5fc?si=U6gX5T9xMQrfMt_P)
