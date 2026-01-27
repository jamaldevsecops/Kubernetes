# 🔐 Traefik TLSOption – Beginner to Production Guide

This guide explains **TLSOption** clearly so new learners understand **TLS, HTTPS, and mTLS** in Traefik.

---

## 🌱 What is TLSOption?

**TLSOption defines HOW TLS behaves.**

It controls:
- 🔒 TLS versions
- 🔑 Cipher suites
- 🪪 Client certificate verification (mTLS)

It does **NOT**:
- Choose certificates
- Do routing

---

## 🧠 Where TLSOption Fits

```
Client
  ↓
EntryPoint (websecure)
  ↓
IngressRoute
  ↓
TLS
   ├─ TLSStore   → which certificate
   ├─ TLSOption  → how strict TLS is
  ↓
Service → Pod
```

---

## 🔐 Basic TLSOption

### When to use
- Internal apps
- Standard HTTPS

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: default-tls
spec:
  minVersion: VersionTLS12
```

---

## 🛡️ Strict TLSOption (Production)

### When to use
- Public apps
- Compliance needs

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: strict-tls
spec:
  minVersion: VersionTLS12
  maxVersion: VersionTLS13
```

---

## 🔐 mTLS (Client → Traefik)

### When to use
- Zero‑trust environments
- Internal platforms

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

---

## 🧩 Client Auth Modes

| Mode | Description |
|-----|-------------|
| ❌ NoClientCert | No client certificate |
| ⚠️ VerifyClientCertIfGiven | Optional cert |
| ✅ RequireAndVerifyClientCert | Mandatory cert |

---

## 🔍 TLSOption vs TLSStore (Very Common Confusion)

| Feature              | TLSOption | TLSStore |
|----------------------|-----------|----------|
| Controls certificate | ❌        | ✅       |
| Controls TLS rules   | ✅        | ❌       |
| Used for mTLS        | ✅        | ❌       |
| Default certificate  | ❌        | ✅       |

---

## 🚨 Common Mistakes

❌ Expecting TLSOption to provide certs  
❌ Weak TLS versions  
❌ Wrong namespace  
❌ Missing CA secret  

---

## ✅ Summary

- TLSOption = TLS policy
- TLSStore = certificate source
- IngressRoute = routing

---

Happy learning 🔐
