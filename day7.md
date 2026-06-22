# Kubernetes Scheduling Labs on AKS (Deployment Based)

## Lab Environment

AKS Cluster with 3 Worker Nodes

| Node     | Label       |
| -------- | ----------- |
| worker-1 | env=dev     |
| worker-2 | env=staging |
| worker-3 | env=prod    |

---

# Initial Setup

## Verify Nodes

```bash
kubectl get nodes
kubectl get nodes --show-labels
```

## Apply Labels

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

# LAB 1 - Node Selector

## node-selector.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dev
spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx-dev

  template:
    metadata:
      labels:
        app: nginx-dev

    spec:
      nodeSelector:
        env: dev

      containers:
      - name: nginx
        image: nginx
```

Deploy

```bash
kubectl apply -f node-selector.yaml
```

Verify

```bash
kubectl get pods -o wide
```

Expected:

```text
All replicas on worker-1
```

---

## Simulation

Remove Label

```bash
kubectl label node worker-1 env-
```

Check Pods

```bash
kubectl get pods -o wide
```

Result:

```text
Pods continue running
```

Reason:

```text
Node Selector is evaluated only during scheduling.
```

---

## Rollout Restart

```bash
kubectl rollout restart deployment nginx-dev
```

Check:

```bash
kubectl get pods
```

Result:

```text
Pods remain Pending
```

---

# LAB 2 - Required Node Affinity

## required-node-affinity.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: prod-app

  template:
    metadata:
      labels:
        app: prod-app

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

Deploy

```bash
kubectl apply -f required-node-affinity.yaml
```

Verify

```bash
kubectl get pods -o wide
```

Expected:

```text
All replicas on worker-3
```

---

## Simulation

Remove Label

```bash
kubectl label node worker-3 env-
```

Pods remain running.

Restart Deployment

```bash
kubectl rollout restart deployment prod-app
```

Result

```text
Pods Pending
```

Reason

```text
requiredDuringSchedulingIgnoredDuringExecution
```

---

# LAB 3 - Preferred Node Affinity

## preferred-node-affinity.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: monitoring-app

  template:
    metadata:
      labels:
        app: monitoring-app

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

Deploy

```bash
kubectl apply -f preferred-node-affinity.yaml
```

Expected:

```text
Prefer worker-2
```

---

## Simulation

Remove Label

```bash
kubectl label node worker-2 env-
```

Restart

```bash
kubectl rollout restart deployment monitoring-app
```

Result

```text
Pods scheduled on any available node
```

Reason

```text
Soft Rule
```

---

# LAB 4 - Node Anti-Affinity

## node-anti-affinity.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-dev-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: no-dev-app

  template:
    metadata:
      labels:
        app: no-dev-app

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

Expected

```text
Pods only on worker-2 and worker-3
```

Never:

```text
worker-1
```

---

# LAB 5 - Pod Affinity

## Backend Deployment

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

Deploy

```bash
kubectl apply -f backend.yaml
```

---

## frontend-affinity.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend

spec:
  replicas: 3

  selector:
    matchLabels:
      app: frontend

  template:
    metadata:
      labels:
        app: frontend

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

Deploy

```bash
kubectl apply -f frontend-affinity.yaml
```

Verify

```bash
kubectl get pods -o wide
```

Expected

```text
Frontend replicas scheduled on backend node
```

---

## Simulation

Delete Backend Pod

```bash
kubectl delete pod <backend-pod-name>
```

Observe

```bash
kubectl get pods -o wide
```

Result

```text
Frontend remains running
```

Reason

```text
IgnoredDuringExecution
```

---

# LAB 6 - Pod Anti-Affinity

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

Deploy

```bash
kubectl apply -f pod-anti-affinity.yaml
```

Expected

```text
worker-1 -> web-pod-1
worker-2 -> web-pod-2
worker-3 -> web-pod-3
```

---

## Simulation

Scale

```bash
kubectl scale deployment web --replicas=4
```

Result

```text
3 Running
1 Pending
```

Reason

```text
Only 3 nodes exist.
```

---

# LAB 7 - Taints

Apply Taint

```bash
kubectl taint node worker-3 env=prod:NoSchedule
```

Verify

```bash
kubectl describe node worker-3
```

---

## Deployment Without Toleration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: regular-app

spec:
  replicas: 5

  selector:
    matchLabels:
      app: regular-app

  template:
    metadata:
      labels:
        app: regular-app

    spec:
      containers:
      - name: nginx
        image: nginx
```

Deploy

```bash
kubectl apply -f regular-app.yaml
```

Expected

```text
Pods avoid worker-3
```

---

# LAB 8 - Tolerations

## toleration.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-toleration

spec:
  replicas: 5

  selector:
    matchLabels:
      app: prod-toleration

  template:
    metadata:
      labels:
        app: prod-toleration

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

Deploy

```bash
kubectl apply -f toleration.yaml
```

Result

```text
Pods MAY run on worker-3
```

Important:

```text
Toleration grants permission.
It does not force placement.
```

---

# LAB 9 - Dedicated Production Nodes

## dedicated-prod.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dedicated-prod

spec:
  replicas: 3

  selector:
    matchLabels:
      app: dedicated-prod

  template:
    metadata:
      labels:
        app: dedicated-prod

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

Expected

```text
All replicas on worker-3
```

---

# LAB 10 - NoExecute Taint

Apply

```bash
kubectl taint node worker-3 env=prod:NoExecute
```

Observe

```bash
kubectl get pods -o wide
```

Result

```text
Pods without toleration are evicted.
```

---

## NoExecute Toleration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: noexecute-demo
spec:
  replicas: 3

  selector:
    matchLabels:
      app: noexecute-demo

  template:
    metadata:
      labels:
        app: noexecute-demo

    spec:

      tolerations:
      - key: "env"
        operator: "Equal"
        value: "prod"
        effect: "NoExecute"

      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Behavior

```text
Pod survives for 60 seconds
Then gets evicted
```
- First apply the file
- Then remove the taint. if this is not removed it will prevent the pod from getting schedule.
```
kubectl taint node worker-3 env=prod:NoSchedule-
```

---

# Useful Commands

```bash
kubectl get nodes --show-labels

kubectl get pods -o wide

kubectl describe node worker-3

kubectl describe pod <pod-name>

kubectl rollout restart deployment <deployment-name>

kubectl scale deployment <deployment-name> --replicas=5

kubectl taint node worker-3 env=prod:NoSchedule

kubectl taint node worker-3 env=prod:NoSchedule-

kubectl label node worker-1 env=dev

kubectl label node worker-1 env-
```
