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

**Single Pod with one container**

**File: single-pod.yaml**

```
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  labels:
    app: nginx
    tier: front
    version: v1
    env: production
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
# Namespace creation

- First deploy **multipod.yaml**
```
apiVersion: v1
kind: Pod
metadata:
  name: multicontainer-pods
  namespace: test-cbc
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
- Then Create the **namspace.yaml** and redeploy multipod.yaml again.

```
apiVersion: v1
kind: Namespace
metadata:
  name: test-cbc
```

# Duplicate port Pod creation
```
apiVersion: v1
kind: Pod
metadata:
  name: duplicate-pods
  namespace: test-cbc
  labels:
    app: httpd
    tier: frontend-backend
    version: v1
spec:
  containers:
  - name: httpd1
    image: httpd
    ports:
    - containerPort: 80
  - name: httpd2
    image: httpd
    ports:
    - containerPort: 80
```

# Kubectl Essential Commands

### pod creation using ad0hoc command
```
kubectl run apache --image=httpd
```
**Generating the yaml file using Ad-Hoc commands**
```
kubectl run apache2 --image=httpd -o yaml --dry-run=client > apache.yaml
```
**Creating pod by specfing command during the pod creation**
```
kubectl run busybox --image=busybox --command -- sh -c "while true; do echo Hello from BusyBox; sleep 2; done"
```
### INSPECT

**List pods**
```
kubectl get pods
```
**List the pods With node info**
```
kubectl get pods -o wide
```
**Full pod details**
```
kubectl describe pod <pod-name>
```
**View pod logs**
```
kubectl logs <pod-name> -n facebook
```
# DEBUG

**Shell into pod**
```
kubectl exec -it <pod-name> -- /bin/sh
```
# EVENTS
```
kubectl get events
```
# RESOURCE USAGE
```
kubectl top nodes
```
```
kubectl top pods -n test-cbc
```
