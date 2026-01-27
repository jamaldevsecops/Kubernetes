# 🚀 Traefik Deployment Project – End-to-End Example (With HTTPS)

This document demonstrates a **complete Traefik-based application deployment** in Kubernetes,
including **Middleware and HTTPS (custom TLS)**.

---

## 🌱 Project Overview

We deploy a sample application and expose it securely using **Traefik CRDs**.

### Application Details
- **Docker Image:** `jamaldevsecops/myapp:v1.1.1`
- **App Name:** `myapp`
- **Container Port:** `3000`
- **HTTP URL:** `http://myapp.apsis.localnet`
- **HTTPS URL:** `https://myapp.apsis.localnet`

---

## 🧠 Architecture (Mental Model)

```
Client
  ↓
DNS (myapp.apsis.localnet → Traefik External IP)
  ↓
IngressRoute (HTTP / HTTPS)
  ↓
Middleware (redirect, security)
  ↓
TLS (TLSStore + TLSOption)
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
├── tls-secret.yaml
├── tlsoption.yaml
├── tlsstore.yaml
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

## 3️⃣ TLS Secret (Custom Certificate)

Create a Kubernetes TLS secret containing your certificate:

```bash
kubectl create secret tls myapp-tls   --cert=tls.crt   --key=tls.key
```

Or YAML form:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
```

---

## 4️⃣ TLSStore (Default Certificate)

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
spec:
  defaultCertificate:
    secretName: myapp-tls
```

---

## 5️⃣ TLSOption (Secure TLS Policy)

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: strict-tls
spec:
  minVersion: VersionTLS12
```

---

## 6️⃣ Middleware

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

## 7️⃣ IngressRoute (HTTP + HTTPS)

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
  tls:
    options:
      name: strict-tls
```

📌 Certificate is loaded from **TLSStore default**.

---

## 🌐 Access the Application

```
http://myapp.apsis.localnet   → redirects
https://myapp.apsis.localnet  → secure access
```

---

## 🧠 Key Takeaways

- HTTPS is enabled using TLSStore + TLSOption
- HTTP is redirected cleanly
- Middleware and TLS are separated concerns
- IngressRoute wires everything together

---

## ✅ Summary

This project now includes:
- Deployment
- Service
- Middleware
- TLSStore
- TLSOption
- HTTPS-enabled IngressRoute

A **production-grade Traefik deployment**.

---

Happy deploying 🔐🚀