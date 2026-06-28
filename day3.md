# Label Creation
- For creating labels there is no need create YAML file. you can simply add labels either through Ad-Hoc commands or under the metadata section.
- And Labels can be created for any objects that are present in the cluster and for nodes as well.

# Creating Labels for namespace, Pod, Nodes

### Check the labels of the resources
**Nodes**
```
kubectl get nodes --show-labels
```
**Pods**
```
kubectl get pods --show-labels
```
### Create a Namespace
```
kubectl create ns prod
```
### Adding labels
```
kubectl label ns prod app=production
```
### Adding labels during namespace creation

- Here you can get the idea that this namespace is for the preprod environment and for the nginx web application.
```
kubectl create ns preprod --labels env=preprod,app=nginx
```
### Creating a pod with with label using ad hoc command
```
kubectl run nginx --image=nginx -n preprod --labels env=preprod,app=webapp
```
**adding label to an exiting pod.**

- first apply the below file.

**pod.yaml**
```
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  namespace: dev
  labels:
    env: dev
spec:
  containers:
  - name: httpd
    image: httpd
    ports:
    - containerPort: 80
```
- Adding labels to the running pod
```
kubectl label pod webserver tier=frontend -n dev
```
- Overwrite the existing labels
```
kubectl label pod webserver env=production -n dev --overwrite
```
### Removing any label
- Removing labels from namespace
```
kubectl label ns dev env-
```
- Removing Labels from Pod
```
kubectl label pod webserver env-
```
### Deleting resources based on the labels
- Deleting a pod based on the labels
```
kubectl delete pod -l app=nginx -n dev
```
# ReplicaSet

- create **rs.yaml**
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-replica
  namespace: prod
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      role: web
    matchExpressions:
      - key: version
        operator: In
        values: [v1, v2, v3]
  template:
    metadata:
      name: web
      labels:
        role: web
        version: v1
        tier: front
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```
# List and inspect the replicaset
- To list the replicaset
```
kubectl get rs <rs name>
```
- To inspect replicaset
```
kubectl describe rs <rs name>
```
### Update number of replicas
- You can change the number of replicas in the yaml file and re-apply the yaml file
- Or you can use ad-hoc command to update the number of replicas
```
kubectl scale --replicas=5 replicaset/<rs name>
```

# Deployment

**deployment.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: myapp
spec:
  replicas: 3
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
```
## Creating and managing Deployments
- Using Ad-hoc commands
```
kubectl create deployment httpd-deploy --image=httpd
```
- List the deployments
```
kubectl get deploy <deployment name>
```
- Inspecting deployments
```
kubectl describe deployment <deployment name>
```
- Setting New image for deployments
```
kubectl set image deployment/my-nginx nginx=nginx:1.21
```
- Scalling the deployments
```
kubectl scale deployment my-nginx --replicas=5
```
# Init Container

**First apply init-pod.yaml**

In the command args we are running one while loop that will check for Redis service on port 6379 and send the ping request every 10 seconds once it gets the pong response, it will allow the main container to start. So to get the pong request we need to apply redis.yaml file so that the init container gets executed successfully.

```
apiVersion: v1
kind: Pod
metadata:
  name: redis-init-pod
  labels:
    app: initapp  
spec:
  initContainers:
  - name: redis-init
    image: aqualyte/redis-alpine
    args:
    - sh
    - -c
    - while true; do STATUS=$(redis-cli -h redis-service -p 6379 ping); if [ $STATUS = "PONG" ]; then echo "Connected"; break; else echo "Not connected"; fi; sleep 10; done
  containers:
  - name: main-container
    image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  selector:
    app: initapp
  ports:
  - protocol: TCP
    port: 80
  type: NodePort
```
**Then apply redis.yaml**
```
apiVersion: v1
kind: Pod
metadata:
  name: redis-server
  labels:
    role: cache-server
spec:
  containers:
  - name: redis-server
    image: redis
    ports:
    - containerPort: 6379
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    role: redis-service
spec:
  selector:
    role: cache-server
  type: ClusterIP
  ports:
  - port: 6379
```
- First we need to apply init-pod.yaml
```
Kubectl apply -f init-pod.yaml
```
- After that apply the redis.yaml
```
kubectl apply -f redis.yaml
```
# DaemonSet

**ds.yaml**

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    app: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
- List DeamonSet
```
kubectl get ds
```
- Describe DeamonSet
```
kubectl describe ds <deamonset name>
```
### Add worker node
- we need to add one more worker node in our cluster to check whether the deamonset will launch another copy of a pod in that newly added worker node.
- So to add the worker node we can use cli commmand. Once the worker node is added then you can see the pod will be launched in the newly created node as well.

**Check the nodepool name before scalling.**

```
az aks nodepool scale \
  --resource-group k8s-bootcamp \
  --cluster-name k8s-bootcamp \
  --name nodepool1 \
  --node-count 3
```

