# Kubernetes Security -- Detailed Guide

## 1. Overview

Kubernetes security involves protecting cluster infrastructure,
workloads, networking, and data. It requires a layered defense strategy.

------------------------------------------------------------------------

## 2. Infrastructure Security

### Node Security

-   Use minimal OS (e.g., Bottlerocket, COS)
-   Apply regular patches
-   Disable unnecessary services
-   Restrict SSH access
-   Use IAM roles instead of static credentials

### Cluster Hardening

-   Private cluster (no public API exposure)
-   Use firewall rules (Security Groups / NSGs)
-   Isolate etcd

------------------------------------------------------------------------

## 3. Authentication & Authorization

### Authentication

-   X.509 certificates
-   OIDC providers (Okta, Google)
-   Cloud IAM integration

### Authorization (RBAC)

-   Principle of least privilege
-   Avoid wildcard permissions

Example:

``` yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

------------------------------------------------------------------------

## 4. Network Security

### Best Practices

-   Default deny all traffic
-   Use NetworkPolicies
-   Segment workloads by namespace

### Advanced

-   Service mesh (Istio, Linkerd)
-   mTLS between services

------------------------------------------------------------------------

## 5. Pod & Container Security

### Key Controls

-   runAsNonRoot: true
-   readOnlyRootFilesystem: true
-   Drop capabilities
-   Avoid privileged containers

### Pod Security Standards

-   Privileged
-   Baseline
-   Restricted (recommended)

------------------------------------------------------------------------

## 6. Secrets Management

### Best Practices

-   Use Kubernetes Secrets (base64 encoded)
-   Enable encryption at rest
-   External secret managers:
    -   HashiCorp Vault
    -   AWS Secrets Manager

------------------------------------------------------------------------

## 7. API Server Security

-   Disable anonymous access
-   Enable audit logs
-   Use TLS certificates
-   Restrict API access via IP whitelisting

------------------------------------------------------------------------

## 8. Logging & Monitoring

### Tools

-   Prometheus + Grafana
-   ELK Stack
-   Falco (runtime security)

### What to Monitor

-   Unauthorized API access
-   Pod exec activity
-   Privilege escalations

------------------------------------------------------------------------

## 9. Image & Supply Chain Security

-   Scan images (Trivy, Clair)
-   Use signed images (Cosign)
-   Avoid latest tag
-   Use private registries

------------------------------------------------------------------------

## 10. Policy Enforcement

### Tools

-   OPA Gatekeeper
-   Kyverno

### Example Policies

-   Disallow privileged containers
-   Require resource limits
-   Enforce labels

------------------------------------------------------------------------

## 11. Common Threats

-   Container escape
-   Privilege escalation
-   Misconfigured RBAC
-   Exposed dashboards
-   Malicious images

------------------------------------------------------------------------

## 12. Security Checklist

-   Enable RBAC
-   Use Network Policies
-   Scan images
-   Encrypt secrets
-   Enforce Pod Security Standards
-   Enable audit logging
-   Keep cluster updated

------------------------------------------------------------------------

## 13. Architecture Model

Cloud → Cluster → Nodes → Pods → Applications

------------------------------------------------------------------------

## 14. Conclusion

Kubernetes security is not a single feature but a continuous process
involving: - Hardening - Monitoring - Policy enforcement - Automation

Adopt DevSecOps practices for best results.
