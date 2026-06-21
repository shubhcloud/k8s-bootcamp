# Kubernetes Scheduling Labs (AKS)

## Lab Setup

Assume an AKS Cluster with 3 Worker Nodes.

| Node     | Label       |
| -------- | ----------- |
| worker-1 | env=dev     |
| worker-2 | env=staging |
| worker-3 | env=prod    |

---

# Verify Nodes

```bash
kubectl get nodes
```

```bash
kubectl get nodes --show-labels
```

---

# Apply Labels

```bash
kubectl label node worker-1 env=dev
kubectl label node worker-2 env=staging
kubectl label node worker-3 env=prod
```

Verify:

```bash
kubectl get nodes --show-labels
```

---

# Node Selector

## node-selector.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-dev
spec:
  nodeSelector:
    env: dev

  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f node-selector.yaml
```

Verify:

```bash
kubectl get pod -o wide
```

Expected:

```text
nginx-dev -> worker-1
```

---

## Simulation

Remove label:

```bash
kubectl label node worker-1 env-
```

Pod remains running.

Delete pod:

```bash
kubectl delete pod nginx-dev
```

Result:

```text
Pending
```

Reason:

Node Selector is checked only during scheduling.

---

# Required Node Affinity

## required-node-affinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-app

spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values:
            - prod

  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f required-node-affinity.yaml
```

Verify:

```bash
kubectl get pod -o wide
```

Expected:

```text
worker-3
```

---

## Simulation

Remove label:

```bash
kubectl label node worker-3 env-
```

Result:

```text
Pod continues running
```

Delete pod:

```bash
kubectl delete pod prod-app
```

Result:

```text
Pending
```

Reason:

requiredDuringSchedulingIgnoredDuringExecution

---

# Preferred Node Affinity

## preferred-node-affinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-app

spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: env
            operator: In
            values:
            - staging

  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f preferred-node-affinity.yaml
```

Expected:

```text
worker-2 preferred
```

---

## Simulation

Remove label:

```bash
kubectl label node worker-2 env-
```

Delete pod:

```bash
kubectl delete pod monitoring-app
```

Result:

```text
Pod scheduled on any available node
```

Reason:

Preferred = Soft Rule

---

# Node Anti-Affinity

## node-anti-affinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-dev-app

spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: NotIn
            values:
            - dev

  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f node-anti-affinity.yaml
```

Expected:

```text
worker-2 or worker-3
```

---

# Backend Deployment

## backend.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend

spec:
  replicas: 1

  selector:
    matchLabels:
      app: backend

  template:
    metadata:
      labels:
        app: backend

    spec:
      containers:
      - name: nginx
        image: nginx
```

Deploy:

```bash
kubectl apply -f backend.yaml
```

---

# Pod Affinity

## pod-affinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend

spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: backend
        topologyKey: kubernetes.io/hostname

  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f pod-affinity.yaml
```

Verify:

```bash
kubectl get pods -o wide
```

Expected:

```text
frontend and backend on same node
```

---

## Simulation

Delete backend pod:

```bash
kubectl delete pod <backend-pod>
```

Frontend remains running.

Reason:

IgnoredDuringExecution

---

# Pod Anti-Affinity

## pod-anti-affinity.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web

spec:
  replicas: 3

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname

      containers:
      - name: nginx
        image: nginx
```

Deploy:

```bash
kubectl apply -f pod-anti-affinity.yaml
```

Verify:

```bash
kubectl get pods -o wide
```

Expected:

```text
worker-1 -> web-1
worker-2 -> web-2
worker-3 -> web-3
```

---

## Simulation

Scale deployment:

```bash
kubectl scale deployment web --replicas=4
```

Result:

```text
4th pod Pending
```

Reason:

Only 3 nodes available.

---

# Apply Taints

## Taint Production Node

```bash
kubectl taint node worker-3 env=prod:NoSchedule
```

Verify:

```bash
kubectl describe node worker-3
```

---

# Test Taint

## normal-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod

spec:
  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f normal-pod.yaml
```

Result:

```text
Will not schedule on worker-3
```

---

# Toleration

## toleration.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-pod

spec:

  tolerations:
  - key: env
    operator: Equal
    value: prod
    effect: NoSchedule

  containers:
  - name: nginx
    image: nginx
```

Deploy:

```bash
kubectl apply -f toleration.yaml
```

Result:

```text
Allowed to run on worker-3
```

Not guaranteed.

---

# Force Placement Using Toleration + Node Affinity

## dedicated-prod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-workload

spec:

  tolerations:
  - key: env
    operator: Equal
    value: prod
    effect: NoSchedule

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values:
            - prod

  containers:
  - name: nginx
    image: nginx
```

Result:

```text
Must run on worker-3
```

---

# NoExecute Taint

Apply:

```bash
kubectl taint node worker-3 env=prod:NoExecute
```

Pods without matching toleration:

```text
Evicted Immediately
```

---

# NoExecute Toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app

spec:

  tolerations:
  - key: env
    operator: Equal
    value: prod
    effect: NoExecute
    tolerationSeconds: 60

  containers:
  - name: nginx
    image: nginx
```

Behavior:

```text
Runs for 60 seconds
Then evicted
```

---

# Useful Troubleshooting Commands

## View Labels

```bash
kubectl get nodes --show-labels
```

## Add Label

```bash
kubectl label node worker-1 env=dev
```

## Remove Label

```bash
kubectl label node worker-1 env-
```

## View Taints

```bash
kubectl describe node worker-3
```

## Add Taint

```bash
kubectl taint node worker-3 env=prod:NoSchedule
```

## Remove Taint

```bash
kubectl taint node worker-3 env=prod:NoSchedule-
```

## Pod Placement

```bash
kubectl get pods -o wide
```

## Pod Events

```bash
kubectl describe pod <pod-name>
```

## Scheduler Errors

Common Messages:

```text
0/3 nodes are available
node(s) didn't match Pod affinity
node(s) didn't match Pod anti-affinity
node(s) didn't match node selector
node(s) had untolerated taint
```

---

# Important Exam / Interview Questions

### Does removing a node label evict running pods?

No.

Because Kubernetes supports:

```text
requiredDuringSchedulingIgnoredDuringExecution
preferredDuringSchedulingIgnoredDuringExecution
```

not:

```text
requiredDuringSchedulingRequiredDuringExecution
```

---

### Does a toleration force a pod onto a node?

No.

Toleration only grants permission.

Use:

```text
Toleration + Node Affinity
```

or

```text
Toleration + Node Selector
```

to force placement.

---

### Which workloads commonly use taints?

* GPU Nodes
* Database Nodes
* Monitoring Nodes
* Security Appliances
* Dedicated Production Node Pools

---

### Which workloads commonly use Pod Anti-Affinity?

* Frontend replicas
* API replicas
* Stateful applications
* High Availability workloads
