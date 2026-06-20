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

## Step 8: Verify Azure Disk Still Exists

```bash
az disk list -o table
```

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
```
