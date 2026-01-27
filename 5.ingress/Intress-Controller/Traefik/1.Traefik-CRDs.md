# 🧠 Traefik CRDs – Complete Beginner to Production Guide

This document explains **Traefik Custom Resource Definitions (CRDs)** in a **clear, structured, and production-ready way**.  
It is designed so a **new learner can understand Traefik deeply** without confusion.

---

## 🌱 What Are CRDs? (In Simple Words)

A **CRD (Custom Resource Definition)** is Kubernetes’ way of saying:

> “I know how to manage Pods and Services, but Traefik needs extra objects.”

So Traefik adds **new resource types** that Kubernetes doesn’t have by default.

These resources are called **Traefik CRDs**.

---

## 🧠 Why Traefik Uses CRDs

Traefik CRDs allow:

- Clean, readable routing rules
- Reusable security logic
- Advanced TLS and mTLS
- Backend trust control
- Dynamic updates (no reloads)

👉 This is why Traefik feels **cloud-native**.

---

## 🧩 Core Traefik CRDs (You Must Know These)

Traefik CRDs can be grouped into **five responsibility areas**:

```
Traffic Entry     → EntryPoints
Traffic Routing   → IngressRoute
Traffic Behavior  → Middleware
TLS & Security    → TLSOption, TLSStore
Backend Trust     → ServersTransport
```

---

## 1️⃣ IngressRoute – Traffic Brain 🧠

### What it does
IngressRoute defines **how requests are matched and routed**.

Controls:
- Host rules
- Path rules
- EntryPoints
- Middleware attachment
- TLS usage
- Backend service selection

### Think of it as:
> 🗺️ The traffic map

```yaml
kind: IngressRoute
```

IngressRoute **always routes to a Kubernetes Service**, never directly to Pods.

---

## 2️⃣ Middleware – Traffic Behavior 🧩

### What it does
Middleware modifies requests **before they reach your app**.

Common use cases:
- 🔁 HTTP → HTTPS redirect
- 🔐 Authentication
- 🛡️ Security headers
- 🚦 Rate limiting
- 🌍 IP whitelist

### Think of it as:
> 🎛️ Traffic filters

```yaml
kind: Middleware
```

⭐ Golden rule:
> **One Middleware = one behavior**

---

## 3️⃣ TLSOption – TLS Rules 🔐

### What it does
TLSOption defines **how strict TLS should be**.

Controls:
- TLS versions
- Cipher suites
- Client certificate verification (mTLS)

Does NOT:
- Provide certificates
- Choose domains

### Think of it as:
> 📜 Security policy

```yaml
kind: TLSOption
```

---

## 4️⃣ TLSStore – Certificate Source 🗂️

### What it does
TLSStore defines **where certificates come from**.

Controls:
- Default TLS certificate
- Fallback certificate behavior

Does NOT:
- Control TLS rules
- Enforce mTLS

### Think of it as:
> 🔑 Certificate vault

```yaml
kind: TLSStore
```

📌 Only **one TLSStore named `default`** per namespace.

---

## 5️⃣ ServersTransport – Backend Trust 🔁

### What it does
ServersTransport defines **how Traefik connects to backend services**.

Controls:
- Backend TLS verification
- Custom CAs
- Backend mTLS
- Connection behavior

Does NOT:
- Affect clients
- Control frontend TLS

### Think of it as:
> 🤝 Trust rules between Traefik and your services

```yaml
kind: ServersTransport
```

---

## 🧠 How CRDs Work Together (BIG PICTURE)

```
Client
  ↓
EntryPoint (web / websecure)
  ↓
IngressRoute
  ↓
Middleware (optional)
  ↓
TLS
   ├─ TLSStore   → certificate
   ├─ TLSOption  → TLS rules
  ↓
ServersTransport (optional)
  ↓
Service
  ↓
Pod
```

This model explains **90% of Traefik behavior**.

---

## 🔍 CRD Responsibility Matrix

| CRD | Routing | Behavior | TLS Rules | Certificates | Backend Trust |
|----|--------|----------|----------|--------------|---------------|
| IngressRoute | ✅ | ❌ | ❌ | ❌ | ❌ |
| Middleware | ❌ | ✅ | ❌ | ❌ | ❌ |
| TLSOption | ❌ | ❌ | ✅ | ❌ | ❌ |
| TLSStore | ❌ | ❌ | ❌ | ✅ | ❌ |
| ServersTransport | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 🚨 Common Beginner Mistakes

❌ Mixing multiple behaviors in one Middleware  
❌ Expecting TLSOption to provide certs  
❌ Using TLSStore for TLS rules  
❌ Forgetting Services (routing to Pods)  
❌ Using ServersTransport for frontend TLS  

---

## 🧠 Mental Shortcuts (Memorize These)

- 🧠 **IngressRoute** → WHERE traffic goes  
- 🧩 **Middleware** → WHAT happens to traffic  
- 🔐 **TLSOption** → HOW TLS behaves  
- 🗂️ **TLSStore** → WHICH certificate  
- 🔁 **ServersTransport** → HOW Traefik trusts backend  

If you remember this, Traefik becomes easy.

---

## ✅ Summary

- Traefik CRDs extend Kubernetes
- Each CRD has a **single responsibility**
- Clean separation = safe production setups
- CRDs combine to form powerful routing & security

---

Happy learning 🚀  
This guide is suitable for **onboarding, production use, and interviews**.
