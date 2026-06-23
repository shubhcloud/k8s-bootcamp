# ClusterIP

### Creating ClusterIP using Ad-hoc commands
- create a pod
```
kubectl run apache --image=httpd --labels app=webapp
```
- Using expose command. Even though if you don't specify the --type it will take clusterip by default.
```  
kubectl expose pod apache --port=80 --name=httpd-clusterip --type=ClusterIP
```
### Creating ClusterIP using YAML files

**clusterip.yaml**
```
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  namespace: prod
  labels:
    env: dev
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: prod
  labels:
    role: web-service
spec:
  selector:
    env: dev
  type: ClusterIP
  ports:
  - port: 8080                  # This is the port that the service will get expose
    targetPort: 80              # This is the port on the pod (containerPort) that the service forwards traffic to
```
- Apply pod.yaml
```
kubectl apply -f clusterip.yaml
```
### Access the application
- First exec into the pod
```
kubectl exec -it webserver -n prod -- sh
```
- Then run the curl command inside from webserver pod
```
curl <svc ip of httpd-clusterip>
```
- You can also run the curl command within the cluster
```
curl <svc ip of httpd-clusterip>
```
- Check the ip of the pod
```
kubectl get pods -o wide
```
- Then check the endpopint of the svc. The pod Ip will be attached with the SVC as endpoint.
```
kubectl get EndpointSlice
```
- For prod namespace as well check endpoint
```
kubectl get EndpointSlice -n prod
```

# NodePort
### Creating NodePort using Ad-hoc commands
- create a pod
```
kubectl run apache --image=httpd --labels app=webapp
```
- Using expose command.
```
kubectl expose pod apache --port=80 --name=httpd-nodeport --type=NodePort
```
### Creating NodePort using YAML files

**nodeport.yaml**
```
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  namespace: nodeport
  labels:
    role: web-service
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: nodeport
  labels:
    role: web-service
spec:
  selector:
    role: web-service
  type: NodePort
  ports:
  - port: 80
    nodePort: 32001
```
- Apply nodeport.yaml
```
kubectl apply -f nodeport.yaml
```
### Access the application
- To access the application, you can copy the public IP of your worker node and run it in a browser with the port.

### Open port in NSG node pool
- create this nsg rule. chneg the node pool name.
```
az network nsg rule create \
  --resource-group MC_k8s-bootcamp_k8s-bootcamp_centralindia \
  --nsg-name aks-agentpool-38176340-nsg \
  --name Allow-NodePort-32001 \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --destination-port-ranges 32001
```

**First delete all the resource of clusterip demo then move forword to Nodeport**

# LoadBalancer

### Creating NodePort using YAML files
```
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  namespace: loadbalancer
  labels:
    role: web-service
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: loadbalancer
  labels:
    role: web-service
spec:
  selector:
    role: web-service
  type: LoadBalancer
  ports:
  - port: 80
```
- Apply loadbalancer.yaml
```
kubectl apply -f loadbalncer.yaml
```
### Access the application
- For accessing the application you can copy the load balancer dns and run it in the browser.

# Network Policy

### Create Pod A

**pod-ns-a.yaml**
```
apiVersion: v1
kind: Namespace
metadata:
  name: namespace-a
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-a
  namespace: namespace-a
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
### Create Pod B

**pod-ns-b.yaml**
```
apiVersion: v1
kind: Namespace
metadata:
  name: namespace-b
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-b
  namespace: namespace-b
  labels:
    app: backend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
### Create Pod C

**pod-ns-c.yaml**
```
apiVersion: v1
kind: Namespace
metadata:
  name: namespace-c
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-db
  namespace: namespace-c
  labels:
    app: db
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-microservice
  namespace: namespace-c
  labels:
    app: microservice
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
### Create Network Policy
**np.yaml**
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-deny-egress
  namespace: namespace-a
spec:
  podSelector: {} # Select all pods in Namespace A
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: namespace-b # Allow traffic from all pods in namespace B
    - podSelector:
        matchLabels:
          app: microservice # Allow traffic from a specific pod in namespace C
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: namespace-c # Namespace C
  egress: []
```
### Test connection
```
curl <pod ip>
```
