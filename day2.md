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
