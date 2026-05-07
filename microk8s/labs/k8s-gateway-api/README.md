# Kubernetes Gateway API + NGINX Gateway Fabric + MetalLB Lab on MicroK8s

This lab demonstrates a complete modern Kubernetes networking setup using:

- MicroK8s
- MetalLB
- Kubernetes Gateway API
- NGINX Gateway Fabric
- HTTP Echo Applications
- Host-based Routing
- External IP exposure

This is a fully local production-style Kubernetes networking lab.

# Lab Architecture

```text id="9by6ae"
                    Browser / curl
                           |
        -----------------------------------------
        |                 |                    |
   red.local         blue.local          green.local
                           |
                    MetalLB External IP
                           |
                 NGINX Gateway Fabric
                           |
                        Gateway
                           |
        ------------------------------------------------
        |                     |                       |
     HTTPRoute            HTTPRoute              HTTPRoute
        |                     |                       |
      red svc              blue svc               green svc
        |                     |                       |
    echo pods             echo pods              echo pods
```



# Lab Objectives

By the end of this lab you will learn:

- Install MicroK8s
- Configure MetalLB
- Install Gateway API CRDs
- Install NGINX Gateway Fabric
- Deploy echo-http applications
- Create Gateway resources
- Configure HTTPRoutes
- Test host-based routing
- Understand modern Kubernetes traffic management

# Pre-Requisites

Minimum VM:

| Resource | Recommended |
| -------- | ----------- |
| CPU      | 2 vCPU      |
| RAM      | 4 GB        |
| Disk     | 20 GB       |

OS:

* Ubuntu 22.04 / 24.04

# Step 1: Install MicroK8s

Install MicroK8s:

```bash id="dx4azt"
sudo snap install microk8s --classic
```

Add current user:

```bash id="n84uyv"
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

Verify:

```bash id="4l3xb7"
microk8s status --wait-ready
```

# Step 2: Enable Required Addons

Enable DNS:

```bash id="9vc0cw"
microk8s enable dns
```

Enable Helm:

```bash id="5l1mbx"
microk8s enable helm3
```

Enable MetalLB:

```bash id="s0r4v6"
microk8s enable metallb
```

You will be asked for IP range.

Example:

```text id="n14exm"
192.168.1.240-192.168.1.250
```

IMPORTANT:

Choose free IPs from your LAN subnet.

# Step 3: Configure kubectl Alias

```bash id="2a0f0k"
alias kubectl='microk8s kubectl'
```

Persist:

```bash id="h2axjm"
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash id="7b6xhy"
kubectl get nodes
```

# Step 4: Install Gateway API CRDs

Install Gateway API CRDs:

```bash id="rk4q2l"
kubectl apply -f \
https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

Verify:

```bash id="5mwl9d"
kubectl get crd | grep gateway
```

Expected:

```text id="qhts8k"
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
```

---

# Step 5: Install NGINX Gateway Fabric

Install NGINX Gateway Fabric:

```bash id="7rmgn8"
helm install nginx-gateway \
oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric \
--version 1.2.0 \
--create-namespace \
-n nginx-gateway
```

Verify:

```bash id="j0ubuo"
kubectl get pods -n nginx-gateway
```

Expected:

```text id="pqgz1f"
nginx-gateway-fabric-xxxxx   Running
```

---

# Step 6: Verify Gateway Service

```bash id="qk9g9s"
kubectl get svc -n nginx-gateway
```

Expected:

```text id="v0i89j"
NAME                                 TYPE           EXTERNAL-IP
nginx-gateway-nginx-gateway-fabric   LoadBalancer   192.168.1.240
```

MetalLB automatically assigns external IP.

---

# Step 7: Deploy Echo Applications

We will use:

```text id="jlwm201"
hashicorp/http-echo
```

This image returns custom text responses, making it perfect for:

* Red / Blue / Green demos
* Gateway API labs
* Ingress labs
* Traffic routing demonstrations


---
# Red Deployment + Service

```yaml id="jlwm301"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: red
  labels:
    app: demo
    version: red

spec:
  replicas: 2

  selector:
    matchLabels:
      app: demo
      version: red

  template:
    metadata:
      labels:
        app: demo
        version: red

    spec:
      containers:
      - name: red-container
        image: hashicorp/http-echo:0.2.3

        args:
        - "-text=Hello from RED Deployment"
        - "-listen=:80"

        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: red-service

spec:
  selector:
    app: demo
    version: red

  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash id="’wini302"
kubectl apply -f red.yaml
```

---

# Blue Deployment + Service

```yaml id="’wini303"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue
  labels:
    app: demo
    version: blue

spec:
  replicas: 2

  selector:
    matchLabels:
      app: demo
      version: blue

  template:
    metadata:
      labels:
        app: demo
        version: blue

    spec:
      containers:
      - name: blue-container
        image: hashicorp/http-echo:0.2.3

        args:
        - "-text=Hello from BLUE Deployment"
        - "-listen=:80"

        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: blue-service

spec:
  selector:
    app: demo
    version: blue

  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash id="’wini304"
kubectl apply -f blue.yaml
```

---

# Green Deployment + Service

```yaml id="’wini305"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green
  labels:
    app: demo
    version: green

spec:
  replicas: 2

  selector:
    matchLabels:
      app: demo
      version: green

  template:
    metadata:
      labels:
        app: demo
        version: green

    spec:
      containers:
      - name: green-container
        image: hashicorp/http-echo:0.2.3

        args:
        - "-text=Hello from GREEN Deployment"
        - "-listen=:80"

        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: green-service

spec:
  selector:
    app: demo
    version: green

  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash id="’wini306"
kubectl apply -f green.yaml
```

---

# Verify Resources

```bash id="’wini307"
kubectl get deploy,svc,pods
```

Expected:

```text id="’wini308"
red-service
blue-service
green-service

red pods
blue pods
green pods
```

---

# Quick Internal Testing

## Red

```bash id="’wini309"
kubectl port-forward svc/red-service 8080:80
```

Open:

```text id="’wini310"
http://localhost:8080
```

Expected:

```text id="’wini311"
Hello from RED Deployment
```

---

## Blue

```bash id="’wini312"
kubectl port-forward svc/blue-service 8081:80
```

Expected:

```text id="’wini313"
Hello from BLUE Deployment
```

---

## Green

```bash id="’wini314"
kubectl port-forward svc/green-service 8082:80
```

Expected:

```text id="’wini315"
Hello from GREEN Deployment
```

---
# Step 8: Verify Deployments

```bash id="pcx9l1"
kubectl get deploy,svc,pods
```

---

# Step 9: Verify GatewayClass

```bash id="x65mzb"
kubectl get gatewayclass
```

Expected:

```text id="of8w4w"
NAME    CONTROLLER
nginx   gateway.nginx.org/nginx-gateway-controller
```

---

# Step 10: Create Gateway

Create gateway.yaml

```yaml id="shk08j"
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: microk8s-gateway
spec:
  gatewayClassName: nginx

  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.local"
```

Apply:

```bash id="vgp3lc"
kubectl apply -f gateway.yaml
```

Verify:

```bash id="1v0jlm"
kubectl get gateway
```

---

# Step 11: Create HTTPRoutes

Create routes.yaml

```yaml id="v3bz7m"
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: red-route
spec:
  parentRefs:
  - name: microk8s-gateway

  hostnames:
  - "red.local"

  rules:
  - backendRefs:
    - name: red
      port: 80

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: blue-route
spec:
  parentRefs:
  - name: microk8s-gateway

  hostnames:
  - "blue.local"

  rules:
  - backendRefs:
    - name: blue
      port: 80

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: green-route
spec:
  parentRefs:
  - name: microk8s-gateway

  hostnames:
  - "green.local"

  rules:
  - backendRefs:
    - name: green
      port: 80
```

Apply:

```bash id="mb5xq7"
kubectl apply -f routes.yaml
```

---

# Step 12: Verify HTTPRoutes

```bash id="k5kcrr"
kubectl get httproute
```

Detailed verification:

```bash id="9dx4cw"
kubectl describe httproute red-route
```

Expected:

```text id="0v34e2"
Accepted: True
ResolvedRefs: True
```

---

# Step 13: Configure Local DNS

Get MetalLB IP:

```bash id="z1q3ph"
kubectl get svc -n nginx-gateway
```

Example:

```text id="g9cz1m"
192.168.1.240
```

Edit hosts file:

```bash id="jlwm8r"
sudo vim /etc/hosts
```

Add:

```text id="vud89g"
192.168.1.240 red.local
192.168.1.240 blue.local
192.168.1.240 green.local
```

---

# Step 14: Test Applications

## Browser Test

Open:

```text id="g0my0q"
http://red.local
http://blue.local
http://green.local
```

---

## curl Test

```bash id="g0mk3z"
curl http://red.local

curl http://blue.local

curl http://green.local
```

---

# Expected Output

You should see echo-server response:

```json id="tz4o2n"
{
  "host": "red.local",
  "headers": {
    "host": "red.local"
  }
}
```


# Step 15: Gateway Verification Commands

## GatewayClass

```bash id="nr8cxg"
kubectl get gatewayclass
```

## Gateway

```bash id="2ppx7q"
kubectl get gateway
```

## HTTPRoutes

```bash id="thlmfm"
kubectl get httproute
```

## Gateway Pods

```bash id="gsklxz"
kubectl get pods -n nginx-gateway
```

## Gateway Service

```bash id="1hkp5l"
kubectl get svc -n nginx-gateway
```

# Understanding the Flow

```text id="rkl9jc"
Browser Request
      |
      v
MetalLB External IP
      |
      v
NGINX Gateway Fabric
      |
      v
Gateway Listener
      |
      v
HTTPRoute Match
      |
      v
Backend Service
      |
      v
Echo Application Pod
```


# Advanced Labs After This

## Path-Based Routing

Example:

```yaml id="p0tgn5"
matches:
- path:
    type: PathPrefix
    value: /api
```


# Header-Based Routing

```yaml id="u9i3an"
matches:
- headers:
  - name: version
    value: beta
```

# Canary Deployments

```yaml id="v2h7v2"
backendRefs:
- name: red
  port: 80
  weight: 90

- name: blue
  port: 80
  weight: 10
```


# TLS / HTTPS

Next step:

* Self-signed certificates
* Cert-manager
* Let's Encrypt


# Cleanup Lab

Delete routes:

```bash id="n32e4w"
kubectl delete -f routes.yaml
```

Delete gateway:

```bash id="mjlwmw"
kubectl delete -f gateway.yaml
```

Delete applications:

```bash id="drmbu8"
kubectl delete deploy red blue green
kubectl delete svc red blue green
```

Delete Gateway Fabric:

```bash id="u2h13t"
helm uninstall nginx-gateway -n nginx-gateway
```


