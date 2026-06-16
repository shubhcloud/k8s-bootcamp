# Create AKS Cluster

### 1. Login to Azure
```
az login
```
### 2. Create Resource Group
```
az group create --name k8s-bootcamp --location centralindia
```
### 3. Create AKS Cluster (2 nodes, cost-effective for lab)
```
az aks create \
  --resource-group k8s-bootcamp \
  --name bootcamp-cluster \
  --node-count 2 \
  --node-vm-size Standard_DS2_v2 \
  --network-policy calico \
  --auto-upgrade-channel none \
  --node-os-upgrade-channel None \
  --generate-ssh-keys
```
### 4. Connect kubectl to the cluster
```
az aks get-credentials --resource-group k8s-bootcamp --name bootcamp-cluster
```
### 5. Verify connection
```
kubectl get nodes
```
# Pod Creation

**Single Pod with multi-container**

**File: Pod.yaml**

```
apiVersion: v1
kind: Pod
metadata:
  name: multicontainer-pods
  namespace: techorp
  labels:
    app: httpd
    tier: frontend-backend
    version: v1
spec:
  containers:
  - name: web
    image: httpd
    ports:
    - containerPort: 80
  - name: db
    image: mysql
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: testcbctest
```
