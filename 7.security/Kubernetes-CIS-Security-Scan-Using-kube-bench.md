# 🛡️ Kubernetes CIS Security Scan Runbook using kube-bench

---

## 🎯 Coverage Scope (Very Important)

kube-bench validates your Kubernetes cluster against the **CIS Kubernetes Benchmark**.

### ✅ What kube-bench actually checks

It audits **node-level configurations**, not workloads.

#### 🔐 Control Plane (Master Node)

* kube-apiserver flags & security settings
* kube-controller-manager configuration
* kube-scheduler configuration
* etcd security (encryption, certs, access)
* Control-plane file permissions

#### 🖥️ Worker Node

* kubelet configuration (auth, TLS, cert rotation)
* kube-proxy configuration
* Node file permissions

#### 📂 Filesystem-level checks

* Static pod manifests (/etc/kubernetes/manifests)
* Systemd service configs
* PKI certificates
* Config files & permissions

---

### ❗ What kube-bench does NOT cover

* ❌ NetworkPolicies enforcement
* ❌ Runtime threats (use Falco)
* ❌ Container image vulnerabilities (use Trivy)
* ❌ RBAC misuse at runtime
* ❌ Application-level security

👉 So kube-bench = **CIS compliance scanner (configuration audit tool)**

---

## 🧠 Understanding cfg Directory (Your Output Explained)

Your structure:

```
cfg/
 ├── cis-1.12
 ├── cis-1.23
 ├── cis-1.24
 ├── eks-*
 ├── gke-*
 ├── aks-*
 ├── k3s-*
 └── rke-*
```

### 🔎 What this means

kube-bench supports **multiple Kubernetes distributions & versions**:

| Directory | Meaning                            |
| --------- | ---------------------------------- |
| cis-1.12  | CIS benchmark for Kubernetes v1.12 |
| cis-1.23  | CIS benchmark for Kubernetes v1.23 |
| cis-1.24  | CIS benchmark for Kubernetes v1.24 |
| eks-*     | AWS EKS specific benchmarks        |
| gke-*     | Google GKE benchmarks              |
| aks-*     | Azure AKS benchmarks               |
| k3s-*     | Lightweight Kubernetes (k3s)       |
| rke/rke2  | Rancher distributions              |

---

### 📌 Important for YOU (kubeadm cluster)

👉 You should use:

```
cis-<your-k8s-version>
```

Example:

| Kubernetes Version | Use cfg                            |
| ------------------ | ---------------------------------- |
| v1.23              | cis-1.23                           |
| v1.24              | cis-1.24                           |
| v1.35              | closest available (e.g., cis-1.24) |

---

### 📂 Inside a CIS directory (Your Example: cis-1.12)

```
cis-1.12/
 ├── config.yaml
 ├── controlplane.yaml
 ├── etcd.yaml
 ├── master.yaml
 ├── node.yaml
 └── policies.yaml
```

### 🔍 Purpose of each file

| File              | Role                                     |
| ----------------- | ---------------------------------------- |
| controlplane.yaml | API server, scheduler, controller checks |
| etcd.yaml         | etcd security checks                     |
| master.yaml       | Master node checks (legacy naming)       |
| node.yaml         | Worker node checks                       |
| policies.yaml     | Test definitions                         |
| config.yaml       | Entry point config                       |

---

### ⚙️ How kube-bench selects benchmark

By default:

✔ Auto-detects Kubernetes version
✔ Maps to closest `cis-*` directory
✔ Executes relevant checks

---

### 🔧 Force a specific benchmark (Recommended for labs)

```bash
sudo ./kube-bench \
  --benchmark cis-1.24 \
  --config-dir `pwd`/cfg
```

---

### 🎯 Real-world Recommendation (Your Environment)

Since you're running **kubeadm (RHEL + CRI-O)**:

✔ Use `cis-*` (NOT eks/gke/aks)
✔ Run on each node separately
✔ Match closest Kubernetes version

---

## 📌 Overview

---

## 📌 Overview

This runbook provides a **production-grade step-by-step guide** to:

* Download kube-bench binary
* Distribute it across nodes
* Execute CIS benchmark scans
* Collect and analyze results

✔ Designed for: **kubeadm-based clusters (RHEL/Ubuntu)**
✔ Covers: **Master (Control Plane) + Worker Nodes**
✔ Works in: **Restricted / Proxy environments**

---

## 🧰 Prerequisites

### System Requirements

* Root or sudo access on all nodes
* SSH access to all nodes
* Internet OR ability to transfer files manually

### Kubernetes

* kubeadm cluster (control-plane + worker nodes)

---

## 📥 Step 1: Download kube-bench (Master Node or Jump Host)

```bash
cd /tmp

KUBE_BENCH_VERSION=0.15.0

curl -LO https://github.com/aquasecurity/kube-bench/releases/download/v${KUBE_BENCH_VERSION}/kube-bench_${KUBE_BENCH_VERSION}_linux_amd64.tar.gz

tar -xvf kube-bench_${KUBE_BENCH_VERSION}_linux_amd64.tar.gz
chmod +x kube-bench
```

📌 Output:

```
kube-bench
cfg/
```

---

## 📦 Step 2: Distribute to All Nodes

### Option A: Using SCP

```bash
scp -r kube-bench cfg root@worker1:/opt/kube-bench/
scp -r kube-bench cfg root@worker2:/opt/kube-bench/
scp -r kube-bench cfg root@master02:/opt/kube-bench/
```

### Option B: Offline (No Internet)

```bash
# From your laptop or jump host
scp kube-bench_*.tar.gz root@node:/tmp/

# On node
cd /opt
mkdir -p kube-bench && cd kube-bench

tar -xvf /tmp/kube-bench_*.tar.gz
```

---

## 📂 Step 3: Prepare Directory Structure (All Nodes)

```bash
mkdir -p /opt/kube-bench
cd /opt/kube-bench
```

Ensure structure:

```
/opt/kube-bench/
 ├── kube-bench
 └── cfg/
```

---

## 🚀 Step 4: Run Scan

### ▶️ On Master Node (Control Plane)

```bash
cd /opt/kube-bench

sudo ./kube-bench \
  --config-dir `pwd`/cfg \
  --config `pwd`/cfg/config.yaml \
  > master-kube-bench-report.txt
```

### ▶️ On Worker Node

```bash
cd /opt/kube-bench

sudo ./kube-bench \
  --config-dir `pwd`/cfg \
  --config `pwd`/cfg/config.yaml \
  > worker-kube-bench-report.txt
```

📌 kube-bench auto-detects:

* Master → control-plane checks
* Worker → node checks

---

## 📊 Step 5: Understand Output

### Sample Output Sections

```
[PASS] 1.1.1 Ensure API server pod spec file permissions
[FAIL] 1.2.3 Ensure etcd encryption is enabled
[WARN] 4.2.6 Ensure kubelet certificate rotation is enabled
```

### Meaning

| Status | Meaning                  |
| ------ | ------------------------ |
| PASS   | Secure configuration     |
| FAIL   | Must fix (security risk) |
| WARN   | Recommended improvement  |
| INFO   | Informational            |

---

## 📁 Step 6: Collect Reports

```bash
# From jump host
scp root@master01:/opt/kube-bench/master-kube-bench-report.txt ./
scp root@worker01:/opt/kube-bench/worker-kube-bench-report.txt ./
scp root@worker02:/opt/kube-bench/worker-kube-bench-report.txt ./
```

---

## 🔍 Step 7: Focus Areas (CKS + Production)

### 🔴 Critical Fixes (High Priority)

* API server flags (authorization, audit)
* etcd encryption
* RBAC enabled
* Kubelet authentication

### 🟡 Medium Priority

* File permissions
* Logging configuration
* TLS settings

### 🟢 Optional

* Benchmark tuning

---

## 🧠 Step 8: Quick Filtering

### Show only FAIL

```bash
grep FAIL master-kube-bench-report.txt
```

### Count failures

```bash
grep -c FAIL master-kube-bench-report.txt
```

---

## 🔁 Step 9: Re-run After Fix

```bash
sudo ./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml
```

✔ Always validate after remediation

---

## ⚙️ Optional: Run via Kubernetes Job

```bash
kubectl apply -f job-master.yaml
kubectl logs <pod-name>
```

✔ Useful when SSH is restricted

---

## 🛑 Common Issues & Fixes

### ❌ Error: permission denied

```bash
sudo ./kube-bench
```

---

### ❌ Error: cfg not found

```bash
Ensure cfg/ exists in same directory
```

---

### ❌ No output / partial checks

✔ Run on correct node type (master vs worker)

---

## 🔐 Security Best Practices

* Run kube-bench regularly (monthly or after upgrade)
* Integrate with CI/CD (Jenkins / pipelines)
* Track results over time
* Fix only **applicable controls** (do not blindly apply all)

---

## 📌 Enterprise Tips (Your Environment)

* Use centralized storage (NFS / Git) for reports
* Combine with:

  * Trivy (image scan)
  * Falco (runtime security)
* Automate via cron or Ansible

---

## 📎 Verification Checklist

✔ kube-bench binary present
✔ cfg directory present
✔ Run as root
✔ Reports generated
✔ FAIL findings reviewed
✔ Fixes validated

---

## 🎯 Final Command (Golden Command)

```bash
sudo ./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml
```

---

## 🚀 Outcome

You will achieve:

* CIS benchmark compliance visibility
* Node-level security posture
* Readiness for **CKS exam + production audit**

---

## 📌 End of Runbook
