# 🚀 Traefik Deployment Project – End-to-End Example

This document demonstrates a **complete Traefik-based application deployment** in Kubernetes.  

---

## 🌱 Project Overview

We will deploy a sample application and expose it using **Traefik CRDs**.

### Application Details
- **Docker Image:** `jamaldevsecops/myapp:v1.1.1`
- **App Name:** `myapp`
- **Container Port:** `3000`
- **Ingress Host:** `myapp.apsis.localnet`

---

## 🧠 Architecture (Mental Model)

```
Client / Browser
  ↓
DNS (myapp.apsis.localnet → Traefik LB IP)
  ↓
Traefik (Ingress Controller)
  ↓
IngressRoute (CRD)
  ↓
Service (ClusterIP)
  ↓
Deployment → Pod (myapp:v1.1.1)
```

---

## 📋 Prerequisites

Before starting, ensure:

- Kubernetes cluster is running
- Traefik is installed and healthy
- DNS entry exists for `myapp.apsis.localnet`
- `kubectl` access to the cluster

---

## 📁 Project Structure

```text
myapp-deployment/
├── deployment.yaml
├── service.yaml
└── ingressroute.yaml
```

---

## 1️⃣ Deployment

Creates Pods running the application container.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: jamaldevsecops/myapp:v1.1.1
          ports:
            - containerPort: 3000
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

## 2️⃣ Service

Exposes the Deployment internally inside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 3000
      targetPort: 3000
```

Apply:
```bash
kubectl apply -f service.yaml
```

Verify:
```bash
kubectl get svc myapp
```

---

## 3️⃣ IngressRoute (Traefik CRD)

Exposes the Service externally using Traefik.

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`myapp.apsis.localnet`)
      kind: Rule
      services:
        - name: myapp
          port: 3000
```

Apply:
```bash
kubectl apply -f ingressroute.yaml
```

---

## 🌐 Access the Application

Open in browser:
```
http://myapp.apsis.localnet
```

Or test via CLI:
```bash
curl http://myapp.apsis.localnet
```

---

## 🔍 Verification & Debugging

Check resources:
```bash
kubectl get deployment myapp
kubectl get svc myapp
kubectl get ingressroute myapp
```

Check Traefik logs:
```bash
kubectl logs -n traefik deploy/traefik
```

---

## 🚨 Common Mistakes

❌ Service port mismatch  
❌ Wrong host name  
❌ IngressRoute missing entryPoints  
❌ DNS not pointing to Traefik  
❌ Traefik not installed  

---

## 🧠 Key Takeaways

- Deployment runs the application
- Service provides stable networking
- IngressRoute exposes the app externally
- Traefik watches CRDs dynamically

---

## 🔜 Next Improvements (Optional)

- Add HTTPS (TLSStore + TLSOption)
- Add Middleware (redirect, auth, rate limit)
- Increase replicas
- Enable monitoring

---

## ✅ Summary

This project demonstrates a **clean, production-style Traefik deployment** using:
- Kubernetes Deployment
- Kubernetes Service
- Traefik IngressRoute CRD

---

Happy deploying 🚀  

