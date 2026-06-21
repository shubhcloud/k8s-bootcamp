# AKS Storage Demo Lab

## Objective

This lab demonstrates:

1. Static Provisioning (Azure Disk → PV → PVC → Deployment)
2. Dynamic Provisioning (StorageClass → PVC → PV → Azure Disk → Deployment)
3. Data Persistence
4. Azure Disk Creation and Binding Process
5. StorageClass Usage in AKS

---

# Demo 1: Static Provisioning

## Architecture

```text
Azure Managed Disk
        ↓
Persistent Volume
        ↓
Persistent Volume Claim
        ↓
Deployment (1 Replica)
```

---

## Step 1: Identify AKS Node Resource Group

```bash
az aks show \
  --resource-group k8s-bootcamp \
  --name k8s-bootcamp \
  --query nodeResourceGroup \
  -o tsv
```

Example Output:

```text
MC_k8s-bootcamp_k8s-bootcamp_centralindia
```

---

## Step 2: Create Azure Managed Disk

```bash
az disk create \
  --resource-group MC_k8s-bootcamp_k8s-bootcamp_centralindia \
  --name demo-pv-disk \
  --size-gb 2 \
  --sku StandardSSD_LRS
```

Get Disk ID:

```bash
az disk show \
  --resource-group MC_k8s-bootcamp_k8s-bootcamp_centralindia \
  --name demo-pv-disk \
  --query id \
  -o tsv
```

Copy the Disk ID.

---

## Step 3: Create Persistent Volume

### pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  csi:
   driver: disk.csi.azure.com
   volumeHandle: /subscriptions/9245f5f4-c6a9-4e3a-a8ed-443681bcdf1a/resourceGroups/MC_k8s-bootcamp_k8s-bootcamp_centralindia/providers/Microsoft.Compute/disks/demo-pv-disk
   fsType: ext4
```

Apply:

```bash
kubectl apply -f pv.yaml
```

Verify:

```bash
kubectl get pv
```

Expected:

```text
NAME      CAPACITY   STATUS
demo-pv   10Gi       Available
```

---

## Step 4: Create PVC

### pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  resources:
    requests:
      storage: 5Gi
  volumeName: demo-pv
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Verify:

```bash
kubectl get pvc
kubectl get pv
```

Expected:

```text
PVC -> Bound
PV  -> Bound
```

---

## Step 5: Create Deployment

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-storage-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: static-storage-demo
  template:
    metadata:
      labels:
        app: static-storage-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: demo-pvc
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get pods
```

---

## Step 6: Write Data

Get Pod Name:

```bash
kubectl get pods
```

Enter Pod:

```bash
kubectl exec -it <pod-name> -- bash
```

Create File:

```bash
echo "Static PV Demo" > /data/demo.txt
cat /data/demo.txt
```

Output:

```text
Static PV Demo
```

---

## Step 7: Demonstrate Persistence

Delete Pod:

```bash
kubectl delete pod <pod-name>
```

Observe:

```bash
kubectl get pods -w
```

Deployment automatically creates a new Pod.

Verify Data:

```bash
kubectl exec -it <new-pod-name> -- cat /data/demo.txt
```

Output:

```text
Static PV Demo
```

---

## Step 8: Make the disk full

- Go inside the pod
```
kubectl exec -it <pod-name> -- bash
```
- And add this file under /data
```
yes "Kubernetes Storage Demo" | head -c 2147483648 > /data/2gb.txt
```
- Check the disk space now. it Should be full
```
df -h
```
- when you attach the azure disk and create the PV from it. During creation kuberntes will not verify the actuall size of the azure disk. so it will simply create the PV even greater then size of the Disk.
- Now the real problem occurs when the mounted directry starts consuming more disk space. Then you cant create more files into that directry. so This mismatch usally happens when you create the PV manually.
- SO to avouid such situationss, you have to plan and based on the application requirement you have to create disk with more disk space. So that this mismatch wont happen.

---

# Demo 2: Dynamic Provisioning

## Architecture

```text
StorageClass
      ↓
PVC
      ↓
PV (Auto Created)
      ↓
Azure Disk (Auto Created)
      ↓
Deployment
```

---

## Step 1: Create Custom StorageClass

### custom-storageclass.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: demo-storageclass
provisioner: disk.csi.azure.com
parameters:
  skuName: StandardSSD_LRS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

Apply:

```bash
kubectl apply -f custom-storageclass.yaml
```

Verify:

```bash
kubectl get sc
```

---

## Step 2: Create PVC

### dynamic-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: demo-storageclass
  resources:
    requests:
      storage: 5Gi
```

Apply:

```bash
kubectl apply -f dynamic-pvc.yaml
```

Check Status:

```bash
kubectl get pvc
```

Expected:

```text
Pending
```

Reason:

```text
WaitForFirstConsumer
```

No Pod is using the PVC yet.

---

## Step 3: Create Deployment

### dynamic-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dynamic-storage-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dynamic-storage-demo
  template:
    metadata:
      labels:
        app: dynamic-storage-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: dynamic-pvc
```

Apply:

```bash
kubectl apply -f dynamic-deployment.yaml
```

---

## Step 4: Observe Automatic Provisioning

Check:

```bash
kubectl get pvc
kubectl get pv
```

Expected:

```text
PVC -> Bound
PV  -> Created Automatically
```

Check Azure:

```bash
az disk list -o table
```

A new Azure Managed Disk is automatically created.

---

## Step 5: Write Data

Get Pod Name:

```bash
kubectl get pods
```

Write File:

```bash
kubectl exec -it <pod-name> -- bash

echo "Dynamic Provisioning Demo" > /data/test.txt

cat /data/test.txt
```

Expected:

```text
Dynamic Provisioning Demo
```

---

## Step 6: Demonstrate Persistence

Delete Pod:

```bash
kubectl delete pod <pod-name>
```

Watch Deployment recreate Pod:

```bash
kubectl get pods -w
```

Verify Data:

```bash
kubectl exec -it <new-pod-name> -- cat /data/test.txt
```

Expected:

```text
Dynamic Provisioning Demo
```

---

# Useful Commands During Demo

## StorageClasses

```bash
kubectl get sc
kubectl describe sc demo-storageclass
```

## Persistent Volumes

```bash
kubectl get pv
kubectl describe pv
```

## Persistent Volume Claims

```bash
kubectl get pvc
kubectl describe pvc
```

## Deployments

```bash
kubectl get deploy
kubectl describe deploy static-storage-demo
kubectl describe deploy dynamic-storage-demo
```

## Pods

```bash
kubectl get pods -o wide
```

## Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Azure Disks

```bash
az disk list -o table
```

---

# Cleanup

## Static Provisioning Cleanup

```bash
kubectl delete deployment static-storage-demo
kubectl delete pvc demo-pvc
kubectl delete pv demo-pv
```

Delete Azure Disk:

```bash
az disk delete \
  --resource-group MC_k8s-bootcamp_k8s-bootcamp_centralindia \
  --name demo-pv-disk \
  --yes
```

---

## Dynamic Provisioning Cleanup

```bash
kubectl delete deployment dynamic-storage-demo
kubectl delete pvc dynamic-pvc
kubectl delete sc demo-storageclass
```

Verify:

```bash
kubectl get pv
kubectl get pvc
kubectl get sc
```

---

# Key Learning Outcomes

1. Difference between PV and PVC.
2. Static vs Dynamic Provisioning.
3. Purpose of StorageClasses.
4. Azure Disk integration with AKS.
5. Data persistence across Pod recreation.
6. WaitForFirstConsumer behavior.
7. Reclaim policies (Retain vs Delete).
8. How Azure Managed Disks are automatically provisioned by AKS.

```
# Kubernetes ConfigMaps and Secrets Demo Lab

# Step 1: Create ConfigMap

## configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "Kubernetes Bootcamp"
  ENVIRONMENT: "Development"
  LOG_LEVEL: "INFO"
  app.properties: |
    app.name=Kubernetes Bootcamp
    environment=Development
    log.level=INFO
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

Verify:

```bash
kubectl get configmap

kubectl describe configmap app-config
```

Expected:

```text
NAME
app-config
```

---

# Step 2: Create Secret

For demo purposes use stringData.

## secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: Password@123
  credentials.txt: |
    username=admin
    password=Password@123
```

Apply:

```bash
kubectl apply -f secret.yaml
```

Verify:

```bash
kubectl get secrets

kubectl describe secret app-secret
```

Expected:

```text
NAME
app-secret
```

---

# Step 3: Create Deployment

This deployment demonstrates:

* ConfigMap as Environment Variable
* ConfigMap as Volume
* Secret as Environment Variable
* Secret as Volume

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-secret-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-secret-demo
  template:
    metadata:
      labels:
        app: config-secret-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_NAME
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: ENVIRONMENT
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_USERNAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        volumeMounts:
        - name: config-volume
          mountPath: /config
        - name: secret-volume
          mountPath: /secrets
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secret-volume
        secret:
          secretName: app-secret
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get deployments

kubectl get pods
```

Expected:

```text
READY   STATUS
1/1     Running
```

---

# Step 4: Verify Environment Variables

Get Pod Name:

```bash
kubectl get pods
```

Example:

```text
config-secret-demo-xxxxx
```

Enter Pod:

```bash
kubectl exec -it <pod-name> -- sh
```

Verify ConfigMap Environment Variables:

```bash
env | grep APP
```

Expected:

```text
APP_NAME=Kubernetes Bootcamp
```

Verify:

```bash
env | grep ENVIRONMENT
```

Expected:

```text
ENVIRONMENT=Development
```

Verify Secret Environment Variables:

```bash
env | grep DB
```

Expected:

```text
DB_USERNAME=admin
DB_PASSWORD=Password@123
```

---

# Step 5: Verify ConfigMap Volume

Inside Pod:

```bash
ls /config
```

Expected:

```text
APP_NAME
ENVIRONMENT
LOG_LEVEL
app.properties
```

View File:

```bash
cat /config/app.properties
```

Expected:

```text
app.name=Kubernetes Bootcamp
environment=Development
log.level=INFO
```

---

# Step 6: Verify Secret Volume

Inside Pod:

```bash
ls /secrets
```

Expected:

```text
DB_PASSWORD
DB_USERNAME
credentials.txt
```

View Secret File:

```bash
cat /secrets/credentials.txt
```

Expected:

```text
username=admin
password=Password@123
```

---

# ConfigMap Update Demo

## Update ConfigMap

```bash
kubectl edit configmap app-config
```

Change:

```text
Development
```

to:

```text
Production
```

Save and exit.

---

## Verify Mounted File

```bash
kubectl exec -it <pod-name> -- cat /config/app.properties
```

Expected:

```text
environment=Production
```

---

## Verify Environment Variable

```bash
kubectl exec -it <pod-name> -- env | grep ENVIRONMENT
```

Expected:

```text
ENVIRONMENT=Development
```

Notice the old value remains.

---

# Important Learning Point

ConfigMap mounted as Volume:

```text
Auto Updates
```

ConfigMap used as Environment Variable:

```text
Requires Pod Restart
```

---

# Demonstrate Environment Variable Refresh

Restart Deployment:

```bash
kubectl rollout restart deployment config-secret-demo
```

Wait:

```bash
kubectl get pods -w
```

Verify:

```bash
kubectl exec -it <new-pod-name> -- env | grep ENVIRONMENT
```

Expected:

```text
ENVIRONMENT=Production
```

---

# Production Troubleshooting Scenario

## Scenario

A deployment suddenly starts failing after a rollout.

Pods show:

```text
CreateContainerConfigError
```

---

# Simulate Failure

Delete ConfigMap:

```bash
kubectl delete configmap app-config
```

Verify Pod:

```bash
kubectl get pods
```

Existing Pod continues running.

Expected:

```text
Running
```

---

# Why Does the Pod Continue Running?

Because:

```text
Pod already started

ConfigMap already mounted

Environment variables already injected
```

No restart has occurred.

---

# Trigger Failure

Delete Running Pod:

```bash
kubectl delete pod <pod-name>
```

Deployment creates a new Pod automatically.

Check:

```bash
kubectl get pods
```

Expected:

```text
CreateContainerConfigError
```

---

# Investigate the Error

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

Expected:

```text
Error: configmap "app-config" not found
```

---

# Troubleshooting Steps

## Step 1

Check Pod Status:

```bash
kubectl get pods
```

Output:

```text
CreateContainerConfigError
```

---

## Step 2

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
configmap not found
```

or

```text
secret not found
```

---

## Step 3

Verify Resource Existence

```bash
kubectl get configmap

kubectl get secret
```

---

## Step 4

Recreate Missing Resource

```bash
kubectl apply -f configmap.yaml
```

or

```bash
kubectl apply -f secret.yaml
```

---

## Step 5

Restart Deployment

```bash
kubectl rollout restart deployment config-secret-demo
```

---

## Step 6

Verify Recovery

```bash
kubectl get pods
```

Expected:

```text
Running
```

---

# Secret Failure Demo

Delete Secret:

```bash
kubectl delete secret app-secret
```

Delete Running Pod:

```bash
kubectl delete pod <pod-name>
```

Verify:

```bash
kubectl get pods
```

Expected:

```text
CreateContainerConfigError
```

Investigate:

```bash
kubectl describe pod <pod-name>
```

Expected:

```text
Error: secret "app-secret" not found
```

---

# Cleanup

Delete Deployment:

```bash
kubectl delete deployment config-secret-demo
```

Delete ConfigMap:

```bash
kubectl delete configmap app-config
```

Delete Secret:

```bash
kubectl delete secret app-secret
```

---

# Key Learning Outcomes

1. What ConfigMaps are.
2. What Secrets are.
3. ConfigMap as Environment Variables.
4. ConfigMap as Mounted Files.
5. Secret as Environment Variables.
6. Secret as Mounted Files.
7. Difference between ConfigMap and Secret.
8. ConfigMap Volume Auto Updates.
9. Environment Variables Require Pod Restart.
10. Understanding CreateContainerConfigError.
11. Troubleshooting Missing ConfigMaps.
12. Troubleshooting Missing Secrets.
13. Real-world Production Rollout Failure Scenario.

