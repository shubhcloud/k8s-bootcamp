# Request and Limits
### CPU Demo

**cpu-demo.yaml**
```
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'sleep 1000']
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
- Apply the file
  ```
  kubectl apply -f cpu-demo.yaml
  ```
### Generate the CPU load
```
kubectl exec -it cpu-demo -- sh -c '
while :; do :; done &
while :; do :; done &
while :; do :; done &
while :; do :; done &
while :; do :; done &
while :; do :; done &
while :; do :; done &
while :; do :; done &
wait
'
```
### Monitor the CPU utilization
- wait for until 1 to 2 mins till the cpu reach its limits
```
watch kubectl top pods
```
### Memory Demo with OOM killed

**memory-demo.yaml**

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
  - name: stress-ctr
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
- Apply the yaml
```
kubectl apply -f memory-demo.yaml
```

# HPA

**php-pod.yaml**
```
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
- Apply the yaml file
```
kubectl apply -f php-pod.yaml
```
### HPA Pod

**hpa.yaml**

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30
```
- Applly HPA
```
kubectl apply -f hpa.yaml
```
### Load Testing
- Once both sample application and hpa is deployed the you can run the below command to generate the load.
```
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```
### List the HPA
- After runing the load generator command. you can list the HPA wit watch command to check live status of cpu utilization getting increased
```
watch kubectl get HPA php-apache
```
# Kubernetes Probes

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
