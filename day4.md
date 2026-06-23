# Deployment Strategies: Recreate, Rolling Update, Blue-Green


## Strategy 1 — Recreate 

**What it does:** Kills ALL old pods first, then creates new pods. Causes downtime.

**When to use:** Dev/staging environments, or apps that can't run multiple versions simultaneously.

**File: `deploy-recreate.yaml`**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recreate-deployment
  labels:
    app: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: my-app-container
        image: nginx:1.21.1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    role: web-service
spec:
  selector:
    app: myapp
  type: NodePort
  ports:
  - port: 80
```

```
kubectl apply -f deploy-recreate.yaml
kubectl get pods -l app=portal-recreate
```

### Now Set new immage for deployment
```
kubectl set image deployment/recreate-deployment my-app-container=nginx:latest
```
### Watch pods — all terminate, then new ones start (DOWNTIME here!)
```
kubectl get pods -l app=portal-recreate -w
```

**🔑 Observation:** All 3 pods terminate at the same time → service unavailable during that gap.

## Strategy 2 — Rolling Update (15–25 min)

**What it does:** Gradually replaces old pods with new ones. Zero downtime.

**When to use:** Most production web apps. Default Kubernetes behavior.

- **MaxSurge** means The number of pod that you want to bring up before terminating the old pods**
- **maxUnavailable** it means  the maximum number of Pods that can be unavailable during the update process.
- **revisionHistoryLimit** means how many old ReplicaSets (revisions) for the Deployment you want to be retained.
- **minReadySeconds** represents the minimum number of seconds in which a newly created Pod has to be ready without crashing or being restarted.

**File: `deploy-rolling.yaml`**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  revisionHistoryLimit: 10
  replicas: 6
  minReadySeconds: 10
  selector:
    matchLabels:
      role: webserver
    matchExpressions:
      - {key: version, operator: In, values: [v1, v2, v3]}
  template:
    metadata:
      name: web
      labels:
        role: webserver
        version: v3
        tier: front
    spec:
      containers:
      - name: web
        image: nginx:1.20-perl
        ports:
        - containerPort: 80
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    role: web-service
spec:
  selector:
    role: webserver
  type: NodePort
  ports:
  - port: 80  
```

```bash
kubectl apply -f deploy-rolling.yaml
```
### Update nginx:1.20-perl → nginx:1.25.5 → nginx:1.26.0 → nginx:latest(rolling)
```
kubectl set image deployment/nginx web=nginx:1.25.5
```
### Watch pods roll one by one
```
kubectl rollout status deployment/nginx
```
### Check rollout history
```
kubectl rollout history deployment/nginx
```
### Annotate the rollout history
```
kubectl annotate deployment/nginx kubernetes.io/change-cause="image updated to 1.25.5"
```
### Rollback to specific version if needed
```
kubectl rollout undo deployment/nginx --to-revision=2
```

**🔑 Observation:** Pods update one at a time → always at least 4 pods serving traffic.

---

## Strategy 3 — Blue-Green Deployment (25–40 min)

**What it does:** Run v1 (Blue) and v2 (Green) simultaneously. Switch traffic instantly by changing the Service selector.

**When to use:** When you need instant rollback ability and can afford double the resources temporarily.

**Files:**

**`deploy-blue.yaml`** (v1 — current production)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-deployment
  labels:
    app: blue-green-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blue
  template:
    metadata:
      labels:
        app: blue
    spec:
      containers:
        - name: blue-container
          image: nileshgule/blue-green-demo:blue
          ports:
          - containerPort: 80
  ```

**`deploy-green.yaml`** (v2 — standby)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-deployment
  labels:
    app: blue-green-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: green
  template:
    metadata:
      labels:
        app: green
    spec:
      containers:
        - name: green-container
          image: nileshgule/blue-green-demo:green
          ports:
          - containerPort: 80
  ```

**`service-bg.yaml`** (points to BLUE initially)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-green-service
  labels:
    app: blue-green-demo
spec:
  type: LoadBalancer
  ports:
    - port: 80
      name: http-port
      protocol: TCP
  selector:
    app: blue
```

```
kubectl apply -f deploy-blue.yaml
kubectl apply -f deploy-green.yaml
kubectl apply -f service-bg.yaml
```

**🔑 Observation:** Zero downtime, instant switch, easy rollback — just one command!

---



## ✅ Comparison Table

| Strategy | Downtime | Rollback | Resource Cost | Best For |
|----------|----------|----------|---------------|----------|
| Recreate | Yes | Fast | Low | Dev environments |
| Rolling Update | None | Slow (undo) | Low | Most prod apps |
| Blue-Green | None | Instant | 2× | Critical systems |
| Canary | None | Instant | Low+ | Feature testing |

---
