# 🚀 Traefik Deployment Project – End-to-End Example (With Middleware)

This document demonstrates a **complete Traefik-based application deployment** in Kubernetes,
including **Middleware integration** in the `IngressRoute`.

---

## 🌱 Project Overview

We deploy a sample application and expose it securely using **Traefik CRDs**.

### Application Details
- **Docker Image:** `jamaldevsecops/myapp:v1.1.1`
- **App Name:** `myapp`
- **Container Port:** `3000`
- **Ingress Host:** `myapp.apsis.localnet`

---

## 🧠 Architecture (Mental Model)

```
Client
  ↓
DNS → Traefik External IP
  ↓
IngressRoute
  ↓
Middleware (redirect, security)
  ↓
Service
  ↓
Deployment → Pod
```

---

## 📁 Project Structure

```text
myapp-deployment/
├── deployment.yaml
├── service.yaml
├── middleware.yaml
└── ingressroute.yaml
```

---

## 1️⃣ Deployment

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

---

## 2️⃣ Service

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

---

## 3️⃣ Middleware

### HTTP → HTTPS Redirect
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: myapp-redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

---

### Security Headers
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: myapp-security-headers
spec:
  headers:
    frameDeny: true
    contentTypeNosniff: true
    browserXssFilter: true
    referrerPolicy: no-referrer
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    stsPreload: true
```

---

## 4️⃣ IngressRoute (With Middleware)

### HTTP IngressRoute (Redirect Only)
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp-http
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`myapp.apsis.localnet`)
      kind: Rule
      middlewares:
        - name: myapp-redirect-https
      services:
        - name: myapp
          port: 3000
```

---

### HTTPS IngressRoute (Secure)
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp-https
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`myapp.apsis.localnet`)
      kind: Rule
      middlewares:
        - name: myapp-security-headers
      services:
        - name: myapp
          port: 3000
```

---

## 🌐 Access the Application

```
http://myapp.apsis.localnet  → redirected
https://myapp.apsis.localnet
```

---

## 🧠 Key Takeaways

- Middleware is **separate** from IngressRoute
- One middleware = one behavior
- HTTP and HTTPS routes are split
- IngressRoute wires everything together

---

## ✅ Summary

This project now includes:
- Deployment
- Service
- Middleware
- IngressRoute

A **clean, production-aligned Traefik setup**.

---

Happy deploying 🚀