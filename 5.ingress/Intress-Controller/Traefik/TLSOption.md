# Traefik TLSOption – Beginner to Production Guide

This document explains **Traefik TLSOption** in a simple, structured way so new learners can understand **TLS, HTTPS, and mTLS** in Traefik without confusion.

---

## 1. What is TLSOption?

**TLSOption defines HOW TLS behaves in Traefik.**

Think of it as:

> "Security rules for HTTPS connections"

TLSOption controls:
- Minimum / maximum TLS versions
- Cipher suites
- Client certificate requirements (mTLS)

It does **not**:
- Select certificates
- Define routing
- Modify requests

---

## 2. Where TLSOption Fits (Mental Model)

```
Client
  ↓
EntryPoint (websecure)
  ↓
IngressRoute
  ↓
TLS
   ├─ TLSStore   → Which certificate?
   ├─ TLSOption  → How strict is TLS?
  ↓
Service → Pod
```

---

## 3. Core Rule (IMPORTANT)

> **TLSOption only defines policy — it does NOT provide certificates**

Certificates come from:
- Kubernetes TLS Secrets
- TLSStore (default cert)

---

## 4. Basic TLSOption (Most Common)

### When to use
- Standard HTTPS
- Internal apps
- Corporate PKI

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: default-tls
spec:
  minVersion: VersionTLS12
```

✔ Blocks TLS 1.0 and 1.1  
✔ Safe default

---

## 5. Strict TLSOption (Production / Compliance)

### When to use
- Public-facing apps
- Compliance (PCI, ISO, SOC2)

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: strict-tls
spec:
  minVersion: VersionTLS12
  maxVersion: VersionTLS13
  cipherSuites:
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

✔ Limits weak crypto  
✔ Strong security posture

---

## 6. mTLS (Client → Traefik)

### What is mTLS?
Normal TLS:
```
Client → verifies Server
```

mTLS:
```
Client ↔ verifies Server
```

### When to use
- Internal platforms
- Admin APIs
- Zero-trust environments

---

### TLSOption with mTLS

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: mtls-required
spec:
  minVersion: VersionTLS12
  clientAuth:
    clientAuthType: RequireAndVerifyClientCert
    secretNames:
      - client-ca
```

📌 `client-ca` must contain:
- `ca.crt` (trusted client CA)

---

## 7. Optional mTLS Modes

| Mode | Behavior |
|---|---|
| NoClientCert | No client cert required |
| VerifyClientCertIfGiven | Optional client cert |
| RequireAndVerifyClientCert | Mandatory client cert |

---

## 8. How to Use TLSOption in IngressRoute

```yaml
tls:
  secretName: myapp-tls
  options:
    name: strict-tls
```

📌 TLSOption must exist in **same namespace**.

---

## 9. TLSOption vs TLSStore (Very Common Confusion)

| Feature | TLSOption | TLSStore |
|---|---|
| Controls cert | ❌ | ✅ |
| Controls TLS rules | ✅ | ❌ |
| Used for mTLS | ✅ | ❌ |
| Default cert | ❌ | ✅ |

---

## 10. Common Mistakes

❌ Expecting TLSOption to provide certs  
❌ Using weak TLS versions  
❌ Forgetting to reference TLSOption  
❌ Wrong namespace  
❌ Missing CA secret for mTLS  

---

## 11. Example: Full Secure HTTPS Setup

```yaml
tls:
  options:
    name: mtls-required
```

- Certificate → TLS Secret or TLSStore
- TLS rules → TLSOption
- Routing → IngressRoute

---

## 12. Summary (Remember This)

- TLSOption = security policy
- TLSStore = certificate source
- IngressRoute = routing
- mTLS = client identity verification

---

Happy learning 🔐  
This guide is suitable for **production, onboarding, and interviews**.
