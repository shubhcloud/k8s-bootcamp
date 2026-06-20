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
