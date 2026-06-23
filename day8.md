# Kubernetes Probes

# Lab Objectives

In this lab you will learn:

* Readiness Probe
* Liveness Probe
* Startup Probe
* How probes fail
* How Kubernetes reacts
* How to troubleshoot probe failures
* How to restore applications back to healthy state

---

# Verify Cluster

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

---

# Section 1 - Readiness Probe

## Goal

Demonstrate:

* Pod becomes Ready
* Pod removed from Service Endpoints
* Pod continues running
* No container restart

---

# Readiness Deployment

File: `readiness.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest

        command:
        - sh
        - -c
        - |
          touch /tmp/ready
          nginx -g 'daemon off;'

        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/ready

          initialDelaySeconds: 5
          periodSeconds: 5
```

---

# Create Service

File: `readiness-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
spec:
  selector:
    app: readiness-demo
  ports:
  - port: 80
    targetPort: 80
```

---

# Deploy

```bash
kubectl apply -f readiness.yaml
kubectl apply -f readiness-svc.yaml
```

---

# Verify

```bash
kubectl get deployments
```

```bash
kubectl get pods
```

Expected:

```text
READY   STATUS
1/1     Running
```

---

# Check Service Endpoints

```bash
kubectl get endpoints readiness-svc
```

Expected:

```text
10.x.x.x:80
```

---

# Simulate Readiness Failure

Find pod:

```bash
kubectl get pods
```

Delete readiness file:

```bash
kubectl exec -it <pod-name> -- rm -f /tmp/ready
```

---

# Observe Failure

```bash
kubectl get pods -w
```

Expected:

```text
READY
0/1
```

Pod status remains:

```text
Running
```

---

# Verify Endpoint Removal

```bash
kubectl get endpoints readiness-svc
```

Expected:

```text
<none>
```

---

# Troubleshooting

## Describe Pod

```bash
kubectl describe pod <pod-name>
```

Expected:

```text
Warning  Unhealthy
Readiness probe failed
```

---

## Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Check Logs

```bash
kubectl logs <pod-name>
```

---

# Restore Readiness

```bash
kubectl exec -it <pod-name> -- touch /tmp/ready
```

---

# Verify Recovery

```bash
kubectl get pods
```

Expected:

```text
1/1 Running
```

---

# Learning Outcome

```text
Readiness Failure
        ↓
Pod Running
        ↓
Traffic Stops
        ↓
No Restart
```

---

# Section 2 - Liveness Probe

## Goal

Demonstrate:

* Application becomes unhealthy
* Kubernetes restarts container
* Self-healing behavior

---

# Liveness Deployment

File: `liveness.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveness-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: liveness-demo
  template:
    metadata:
      labels:
        app: liveness-demo
    spec:
      containers:
      - name: busybox
        image: busybox

        command:
        - sh
        - -c
        - |
          touch /tmp/healthy
          sleep 3600

        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy

          initialDelaySeconds: 5
          periodSeconds: 5
```

---

# Deploy

```bash
kubectl apply -f liveness.yaml
```

---

# Verify

```bash
kubectl get pods
```

---

# Simulate Failure

Delete health file:

```bash
kubectl exec -it <pod-name> -- rm -f /tmp/healthy
```

---

# Observe

```bash
kubectl get pods -w
```

Expected:

```text
RESTARTS
1
```

then

```text
RESTARTS
2
```

---

# Troubleshooting

## Describe Pod

```bash
kubectl describe pod <pod-name>
```

Expected:

```text
Liveness probe failed
Container restarted
```

---

## Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Current Logs

```bash
kubectl logs <pod-name>
```

---

## Previous Logs

```bash
kubectl logs <pod-name> --previous
```

Important:

```text
Current Logs = Running Container

Previous Logs = Crashed Container
```

---

# Verify Recovery

Deployment automatically creates healthy container.

```bash
kubectl get pods
```

Expected:

```text
READY   STATUS   RESTARTS
1/1     Running  1
```

---

# Learning Outcome

```text
Liveness Failure
        ↓
Application Unhealthy
        ↓
Container Restart
        ↓
Self Healing
```

---

# Section 3 - Startup Probe

## Goal

Demonstrate:

* Application startup validation
* Startup probe failures
* CrashLoopBackOff scenario

---

# Startup Deployment

File: `startup.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: startup-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: startup-demo
  template:
    metadata:
      labels:
        app: startup-demo
    spec:
      containers:
      - name: busybox
        image: busybox

        command:
        - sh
        - -c
        - |
          sleep 120

        startupProbe:
          exec:
            command:
            - cat
            - /tmp/started

          periodSeconds: 5
          failureThreshold: 6
```

---

# Deploy

```bash
kubectl apply -f startup.yaml
```

---

# Observe Failure

```bash
kubectl get pods -w
```

Expected:

```text
CrashLoopBackOff
```

---

# Troubleshooting

## Describe Pod

```bash
kubectl describe pod <pod-name>
```

Expected:

```text
Startup probe failed
```

---

## Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Logs

```bash
kubectl logs <pod-name>
```

---

# Fix Deployment

Edit deployment:

```yaml
command:
- sh
- -c
- |
  touch /tmp/started
  sleep 120
```

---

# Apply Again

```bash
kubectl apply -f startup.yaml
```

---

# Verify Recovery

```bash
kubectl rollout status deployment startup-demo
```

```bash
kubectl get pods
```

Expected:

```text
1/1 Running
```

---

# Learning Outcome

```text
Startup Failure
       ↓
Container Never Becomes Healthy
       ↓
CrashLoopBackOff
```

---

# Common Probe Troubleshooting Commands

## Pods

```bash
kubectl get pods
```

```bash
kubectl get pods -w
```

---

## Describe

```bash
kubectl describe pod <pod-name>
```

---

## Logs

```bash
kubectl logs <pod-name>
```

```bash
kubectl logs <pod-name> --previous
```

---

## Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Deployments

```bash
kubectl get deployments
```

```bash
kubectl rollout status deployment/<deployment-name>
```

---

## ReplicaSets

```bash
kubectl get rs
```

---

## Services

```bash
kubectl get svc
```

---

## Endpoints

```bash
kubectl get endpoints
```

---

## Resource Usage

```bash
kubectl top pod
```

---

# Probe Failure Summary

| Probe           | Failure Result                     |
| --------------- | ---------------------------------- |
| Readiness Probe | Traffic Stops                      |
| Liveness Probe  | Container Restart                  |
| Startup Probe   | Startup Failure / CrashLoopBackOff |

---

# Interview Questions

### What happens when Readiness Probe fails?

Pod remains running but stops receiving traffic.

---

### What happens when Liveness Probe fails?

Container is restarted.

---

### What happens when Startup Probe fails?

Application never becomes healthy and may enter CrashLoopBackOff.

---

### Which command is most useful during probe troubleshooting?

```bash
kubectl describe pod <pod-name>
```

Because it shows:

* Probe failures
* Restart counts
* Events
* Container state
