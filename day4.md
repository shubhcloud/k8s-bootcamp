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

# Rolling Update Failure Scenarios

## Objective

In production environments, a Rolling Update does not always succeed. A new version of the application may:

* Crash immediately after startup
* Fail to pull the container image
* Never become Ready

In this lab, we will intentionally break a deployment during a Rolling Update and learn how to:

1. Observe the rollout process.
2. Identify rollout failures.
3. Troubleshoot the root cause.
4. Roll back to the previous working version.
5. Fix the issue and redeploy successfully.

---

# Scenario 1: Rolling Update Causing CrashLoopBackOff

## Step 1: Deploy Healthy Application

Create a file named:

```bash
deployment-v1.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

  selector:
    matchLabels:
      app: rolling-demo

  template:
    metadata:
      labels:
        app: rolling-demo

    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

Deploy:

```bash
kubectl apply -f deployment-v1.yaml
```

Verify:

```bash
kubectl get deploy
kubectl get rs
kubectl get pods
```

Expected:

```text
NAME           READY   UP-TO-DATE   AVAILABLE
rolling-demo   4/4     4            4
```

---

## Step 2: Verify Rollout History

```bash
kubectl rollout history deployment rolling-demo
```

Expected:

```text
REVISION
1
```

---

## Step 3: Deploy Faulty Version

Create:

```bash
deployment-v2-crash.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

  selector:
    matchLabels:
      app: rolling-demo

  template:
    metadata:
      labels:
        app: rolling-demo

    spec:
      containers:
      - name: nginx
        image: nginx:1.27

        command:
        - /bin/sh
        - -c
        - |
          cat /tmp/file-does-not-exist

        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment-v2-crash.yaml
```

---

## Step 4: Observe Rolling Update

Open another terminal:

```bash
kubectl get pods -w
```

Expected:

```text
rolling-demo-xxxxx            Running
rolling-demo-yyyyy            Running
rolling-demo-zzzzz            Running

rolling-demo-newpod           CrashLoopBackOff
```

Notice:

* Old Pods remain available.
* New Pod continuously crashes.
* Deployment rollout cannot continue.

This demonstrates how Rolling Update protects application availability.

---

## Step 5: Check Deployment Status

```bash
kubectl rollout status deployment rolling-demo
```

Possible output:

```text
Waiting for deployment "rolling-demo" rollout to finish
```

After some time:

```text
deployment "rolling-demo" exceeded its progress deadline
```

---

## Step 6: Examine ReplicaSets

```bash
kubectl get rs
```

Example:

```text
NAME                     DESIRED   CURRENT   READY
rolling-demo-abc123      4         4         4
rolling-demo-def456      1         1         0
```

Explanation:

```text
Old ReplicaSet = Healthy
New ReplicaSet = Failing

Deployment cannot replace old pods
until new pods become Ready.
```

---

## Step 7: Troubleshooting CrashLoopBackOff

Find failing Pod:

```bash
kubectl get pods
```

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

Look for events:

```text
Back-off restarting failed container
```

Check logs:

```bash
kubectl logs <pod-name>
```

You may see:

```text
Container exited immediately
```

---

## Step 8: Roll Back Deployment

View history:

```bash
kubectl rollout history deployment rolling-demo
```

Rollback:

```bash
kubectl rollout undo deployment rolling-demo
```

Verify:

```bash
kubectl get pods
```

Expected:

```text
4/4 Running
```

Check rollout:

```bash
kubectl rollout status deployment rolling-demo
```

Expected:

```text
deployment successfully rolled out
```

---

## Step 9: Deploy Fixed Version

Create:

```bash
deployment-v3-fixed.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

  selector:
    matchLabels:
      app: rolling-demo

  template:
    metadata:
      labels:
        app: rolling-demo

    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment-v3-fixed.yaml
```

Monitor:

```bash
kubectl rollout status deployment rolling-demo
```

Expected:

```text
deployment successfully rolled out
```

---

# Scenario 2: Rolling Update Causing ImagePullBackOff

Delete existing deployment:

```bash
kubectl delete deployment rolling-demo
```

---

## Step 1: Deploy Healthy Version

```bash
kubectl apply -f deployment-v1.yaml
```

Verify:

```bash
kubectl get pods
```

All Pods should be Running.

---

## Step 2: Deploy Invalid Image Version

Create:

```bash
deployment-v2-imagepull-error.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

  selector:
    matchLabels:
      app: rolling-demo

  template:
    metadata:
      labels:
        app: rolling-demo

    spec:
      containers:
      - name: nginx
        image: nginx:this-tag-does-not-exist
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment-v2-imagepull-error.yaml
```

---

## Step 3: Watch Rolling Update

```bash
kubectl get pods -w
```

Expected:

```text
Old Pods Running

New Pod Pending

ErrImagePull

ImagePullBackOff
```

---

## Step 4: Check Deployment Status

```bash
kubectl rollout status deployment rolling-demo
```

Output:

```text
Waiting for deployment "rolling-demo" rollout to finish
```

Eventually:

```text
deployment "rolling-demo" exceeded its progress deadline
```

---

## Step 5: Troubleshoot ImagePullBackOff

Find Pod:

```bash
kubectl get pods
```

Describe:

```bash
kubectl describe pod <pod-name>
```

Events section:

```text
Failed to pull image

ErrImagePull

ImagePullBackOff
```

Example:

```text
manifest for nginx:this-tag-does-not-exist not found
```

---

## Step 6: Check ReplicaSets

```bash
kubectl get rs
```

Example:

```text
NAME                     DESIRED   CURRENT   READY
rolling-demo-abc123      4         4         4
rolling-demo-def456      1         1         0
```

Explanation:

```text
Old Pods remain available.

New Pods never start.

Traffic continues to flow
to the old ReplicaSet.

Deployment remains incomplete.
```

---

## Step 7: Fix the Image

Create:

```bash
deployment-v3-image-fixed.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

  selector:
    matchLabels:
      app: rolling-demo

  template:
    metadata:
      labels:
        app: rolling-demo

    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment-v3-image-fixed.yaml
```

Monitor:

```bash
kubectl rollout status deployment rolling-demo
```

Expected:

```text
deployment successfully rolled out
```

---

# Production Troubleshooting Workflow

Whenever a deployment is stuck:

```text
Deployment Failed
        │
        ▼
kubectl rollout status
        │
        ▼
kubectl get pods
        │
        ▼
Pod Healthy?
    /       \
   No        Yes
   │
   ▼
kubectl describe pod
   │
   ▼
Check Events
   │
   ├── ImagePullBackOff
   │       └─ Verify image name and tag
   │
   ├── CrashLoopBackOff
   │       └─ Check logs and startup command
   │
   └── Pending
           └─ Check resources and scheduling
```

## Key Learning

Rolling Updates improve availability because Kubernetes does not immediately terminate all old Pods.

If the new version fails:

* Existing Pods continue serving traffic.
* Deployment rollout pauses.
* Administrators can investigate.
* Rollback can quickly restore a healthy state.

This behavior is one of the primary reasons RollingUpdate is the default deployment strategy in Kubernetes.

