# Traefik Middleware – Beginner to Production Guide

This document explains **Traefik Middleware** in a simple, practical way so new learners can quickly understand and use it safely in production.

---

## 1. What is Middleware?

**Middleware = logic applied to requests BEFORE they reach your application.**

Middleware can:
- Redirect traffic (HTTP → HTTPS)
- Add or modify headers
- Authenticate users
- Limit request rate
- Allow or block IPs

👉 Middleware **does not change your application code**.

---

## 2. Core Rule (VERY IMPORTANT)

> **One Middleware resource can contain ONLY ONE behavior**

❌ This is invalid:
- redirect + headers + rateLimit in one Middleware

✅ Correct:
- Separate Middleware resources
- Chain them in `IngressRoute`

---

## 3. Common Middleware Types

### 3.1 HTTP → HTTPS Redirect

**When to use**
- Always (security baseline)
- Public or internal apps

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

⚠️ Apply ONLY on HTTP (`web`) entryPoint.

---

### 3.2 Security Headers

**When to use**
- Browser-based apps
- APIs exposed externally

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
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

### 3.3 Basic Authentication

**When to use**
- Admin panels
- Dashboards
- Internal tools

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
spec:
  basicAuth:
    secret: myapp-basic-auth
```

Secret must contain htpasswd-formatted users.

---

### 3.4 Rate Limiting

**When to use**
- APIs
- Protect against brute-force or abuse

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 100
    burst: 50
```

Meaning:
- 100 requests/sec average
- Short burst of 50 allowed

---

### 3.5 IP Whitelist

**When to use**
- VPN-only apps
- Office/internal access
- Admin endpoints

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ip-whitelist
spec:
  ipWhiteList:
    sourceRange:
      - 10.0.0.0/8
      - 192.168.1.0/24
      - 203.0.113.10/32
```

---

## 4. Chaining Multiple Middlewares

Attach multiple middlewares in an **IngressRoute**.

```yaml
middlewares:
  - name: ip-whitelist
  - name: rate-limit
  - name: basic-auth
  - name: security-headers
```

### Recommended Order
1. Redirect
2. IP whitelist
3. Rate limit
4. Authentication
5. Headers

---

## 5. Correct HTTP + HTTPS Pattern

### HTTP IngressRoute (Redirect Only)
```yaml
entryPoints:
  - web
middlewares:
  - name: redirect-https
```

### HTTPS IngressRoute (Security)
```yaml
entryPoints:
  - websecure
middlewares:
  - name: ip-whitelist
  - name: rate-limit
  - name: security-headers
```

---

## 6. Common Mistakes

❌ Multiple middleware types in one resource  
❌ Redirect middleware on HTTPS  
❌ Middleware in wrong namespace  
❌ Too aggressive rate limits  
❌ IP whitelist blocking load balancer IPs  

---

## 7. Mental Model (Remember This)

```
Request
  ↓
Redirect?
  ↓
IP allowed?
  ↓
Rate OK?
  ↓
Authenticated?
  ↓
Headers added
  ↓
Service → Pod
```

---

## 8. Summary

- Middleware is reusable traffic logic
- One Middleware = one behavior
- Chain middleware in IngressRoute
- Always split HTTP and HTTPS responsibilities

---

Happy learning 🚀  
