# 📅 DAY 6 — Gateway API: The Modern Alternative to Ingress

## Why Gateway API over Ingress?

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Standard spec | Partial | Full CNCF standard |
| Traffic splitting | Annotations only | Native weights |
| Header-based routing | Vendor-specific | Built-in |
| Role separation | None | GatewayClass/Gateway/Route |
| GRPC/TCP support | No | Yes |
| Canary support | Hacky | Native |

**Three Gateway API resources:**
- **GatewayClass** → Defines which controller manages Gateways (like StorageClass)
- **Gateway** → The actual load balancer / entry point (like an Ingress Controller)
- **HTTPRoute** → Routing rules (like Ingress rules, but much more powerful)

---

## Step 1 — Install Gateway API CRDs

```bash
# Install Envoy Gateway via Helm (OCI registry — no repo add needed)
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.8.0 -n envoy-gateway --create-namespace


# Wait for the controller to be ready
watch kubectl get all -n envoy-gateway

# Envoy Gateway provisions its own LoadBalancer per Gateway — check after Gateway creation
```

---

## Step 2 — Create GatewayClass + Gateway (10 min)

**`gatewayclass.yaml`**
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**`gateway.yaml`**

**Once the gateway.yaml file is aplied then only the external ip will appear**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-gateway
  namespace: envoy-gateway
spec:
  gatewayClassName: envoy-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

```bash
kubectl apply -f gatewayclass.yaml
kubectl apply -f gateway.yaml

# Envoy Gateway creates a managed Envoy proxy pod + LoadBalancer per Gateway
kubectl get gateway -n envoy-gateway
kubectl get pods -n envoy-gateway    # See the managed Envoy proxy pod
kubectl describe gateway envoy-gateway -n envoy-gateway
```

---

## Step 3 — Ensure deployment and Backend Services Exist

Make sure ClusterIP services exist for each version (Gateway API routes to Services):

**`deploy-blue.yaml`** (v1 — current production)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-deployment
  labels:
    app: blue-green-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blue
  template:
    metadata:
      labels:
        app: blue
    spec:
      containers:
        - name: blue-container
          image: nileshgule/blue-green-demo:blue
          ports:
          - containerPort: 80
  ```

**`deploy-green.yaml`** (v2 — standby)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-deployment
  labels:
    app: blue-green-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: green
  template:
    metadata:
      labels:
        app: green
    spec:
      containers:
        - name: green-container
          image: nileshgule/blue-green-demo:green
          ports:
          - containerPort: 80
  ```

**`blue-service.yaml`** (points to BLUE initially)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-service
  labels:
    app: blue-svc
spec:
  ports:
    - port: 80
      name: blue-http-port
      protocol: TCP
  selector:
    app: blue
```

**`green-service.yaml`** (points to BLUE initially)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: green-service
  labels:
    app: green-svc
spec:
  ports:
    - port: 80
      name: green-http-port
      protocol: TCP
  selector:
    app: green
```


---

## Step 4 — HTTPRoute: domain-Based Routing (15 min)

**`domain-route.yaml`**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: blue-route
spec:
  parentRefs:
    - name: envoy-gateway
      namespace: envoy-gateway
  hostnames:
    - blue.cbc.com
  rules:
  - backendRefs:
    - name: blue-service
      port: 80
```
### HTTPRoute: domain-Based Routing(GREEN)

**`path-route.yaml`**
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: green-route
spec:
  parentRefs:
    - name: envoy-gateway
      namespace: envoy-gateway
  hostnames:
    - green.cbc.com
  rules:
  - backendRefs:
    - name: green-service
      port: 80
```

```bash
kubectl apply -f domain-route.yaml path-route.yaml
kubectl get httproute
```

## Step 5 — HTTPRoute: Weight-Based Traffic Splitting (15 min)

This is native canary in Gateway API — no pod-count tricks needed!

**`httproute-weights.yaml`**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: weighted-route
spec:
  parentRefs:
    - name: envoy-gateway
      namespace: envoy-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: blue-service
      port: 80
      weight: 60
    - name: green-service
      port: 80
      weight: 40
```

```bash
kubectl apply -f httproute-weights.yaml

```


