# 🛡️ Kubernetes Security & Compliance Runbook using Kubescape

---

## 🎯 Coverage Scope (Very Important)

Kubescape is a **Kubernetes security platform** that combines:

👉 Configuration scanning + RBAC analysis + attack path detection

---

### ✅ What Kubescape checks

#### 🔐 CIS & NSA Compliance

* CIS Kubernetes Benchmark
* NSA Hardening Guide
* MITRE ATT&CK mapping

#### ⚙️ Kubernetes Resources

* Deployments, Pods, Services
* RBAC (Roles, RoleBindings, ClusterRoles)
* Secrets & ConfigMaps

#### 🧠 Misconfiguration Detection

* Privileged containers
* HostPath mounts
* Running as root
* Missing resource limits

#### 🔗 Attack Path Analysis

* Identifies how an attacker can move laterally
* Shows privilege escalation paths

---

### ❗ What Kubescape does NOT cover

* ❌ Runtime threat detection (use Falco)
* ❌ Deep network probing (use kube-hunter)

👉 Kubescape = **Cluster-wide security posture + misconfiguration scanner**

---

## 🧰 Prerequisites

* kubectl configured
* Access to cluster
* Internet OR offline bundle

---

## 📥 Installation

### Option 1: Install CLI (Recommended)

```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
```

Verify:

```bash
kubescape version
```

---

### Option 2: Download binary (offline)

```bash
curl -LO https://github.com/kubescape/kubescape/releases/latest/download/kubescape-linux-amd64
chmod +x kubescape-linux-amd64
mv kubescape-linux-amd64 /usr/local/bin/kubescape
```

---

## 🚀 Basic Scanning

### Scan entire cluster

```bash
kubescape scan
```

---

### Scan specific namespace

```bash
kubescape scan ns opsnav
```

---

### Scan specific framework (CIS)

```bash
kubescape scan framework cis
```

---

### Scan NSA framework

```bash
kubescape scan framework nsa
```

---

## 📊 Output Understanding

### Sample

```
Control: Ensure privileged containers are not allowed
Status: Failed
Severity: High
```

---

### Status Meaning

| Status  | Meaning          |
| ------- | ---------------- |
| Passed  | Secure           |
| Failed  | Misconfiguration |
| Skipped | Not applicable   |

---

## 🔍 Common Findings → Remediation

### 1) Privileged container

#### Risk

* Full host access

#### Fix

```yaml
securityContext:
  privileged: false
```

---

### 2) Running as root

#### Fix

```yaml
securityContext:
  runAsNonRoot: true
```

---

### 3) No resource limits

#### Fix

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

---

### 4) Over-permissive RBAC

#### Fix

* Avoid `cluster-admin`
* Use least privilege roles

---

### 5) HostPath usage

#### Risk

* Host filesystem exposure

#### Fix

* Avoid HostPath unless required

---

## 🔁 Save Reports

```bash
kubescape scan --format json --output results.json
```

---

## 🤖 Automation Script

```bash
cat >/usr/local/bin/run-kubescape.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

OUT_DIR=/var/log/kubescape
mkdir -p "$OUT_DIR"
TS=$(date +%F-%H%M%S)

kubescape scan framework cis \
  --format json \
  --output "$OUT_DIR/kubescape-${TS}.json"
EOF

chmod +x /usr/local/bin/run-kubescape.sh
```

---

## 🔁 Jenkins Integration

```groovy
stage('Kubescape Scan') {
  steps {
    sh '''
      kubescape scan framework cis --format json --output kubescape.json || true
    '''
    archiveArtifacts artifacts: 'kubescape.json', fingerprint: true
  }
}
```

---

## 🔐 Enterprise Best Practices

* Run after every deployment
* Integrate with CI/CD
* Track trends over time
* Combine with:

  * kube-bench
  * kube-hunter
  * Trivy

---

## 📎 Verification Checklist

✔ kubescape installed
✔ cluster scanned
✔ findings reviewed
✔ fixes applied
✔ reports stored

---

## 🎯 Golden Command

```bash
kubescape scan framework cis
```

---

## 🚀 Outcome

* Full cluster visibility
* Misconfiguration detection
* Compliance readiness (CIS/NSA)

---

## 📌 End of Runbook
