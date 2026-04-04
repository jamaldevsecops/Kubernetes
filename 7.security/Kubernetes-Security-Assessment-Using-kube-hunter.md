# 🕵️ Kubernetes Security Assessment Runbook using kube-hunter

---

## 🎯 Coverage Scope (Very Important)

kube-hunter is a **Kubernetes penetration testing tool**.

👉 Unlike kube-bench (configuration audit), kube-hunter focuses on:

### ✅ What kube-hunter checks

#### 🌐 Network Exposure

* Open Kubernetes ports (API server, kubelet, etcd)
* Publicly accessible endpoints
* Node IP exposure

#### 🔓 Security Weaknesses

* Anonymous access enabled
* Unauthenticated kubelet access
* Exposed dashboards
* Insecure API server flags

#### 🧠 Attack Simulation

* Simulates attacker behavior
* Attempts to access cluster components
* Identifies lateral movement paths

#### 🔍 Discovery

* Nodes, services, endpoints
* Kubernetes version detection
* Cluster topology mapping

---

### ❗ What kube-hunter does NOT cover

* ❌ CIS benchmark compliance (use kube-bench)
* ❌ Image vulnerabilities (use Trivy)
* ❌ Runtime detection (use Falco)
* ❌ Internal RBAC misconfiguration (limited)

👉 kube-hunter = **Active security probing tool (attacker perspective)**

---

## 🧰 Prerequisites

* Python 3 installed OR Docker available
* Network access to cluster (internal or external)
* Root/sudo (optional but recommended)

---

## 📥 Option 1: Install via pip (Recommended for your environment)

```bash
pip3 install kube-hunter
```

Verify:

```bash
kube-hunter --help
```

---

## 📥 Option 2: Run via Docker (Best for isolated scan)

```bash
docker run --rm aquasec/kube-hunter
```

---

## 📥 Option 3: Download binary (offline friendly)

```bash
cd /tmp

curl -LO https://github.com/aquasecurity/kube-hunter/releases/latest/download/kube-hunter-linux-amd64

chmod +x kube-hunter-linux-amd64
mv kube-hunter-linux-amd64 /usr/local/bin/kube-hunter
```

---

## 🚀 Scanning Modes (Very Important)

### 1️⃣ Passive Scan (Safe)

✔ No active exploitation
✔ Only detects exposures

```bash
kube-hunter --remote <cluster-ip>
```

---

### 2️⃣ Active Scan (Recommended for internal testing)

✔ Performs active probing
✔ Simulates attacks

```bash
kube-hunter --remote <cluster-ip> --active
```

---

### 3️⃣ Internal Scan (Run inside cluster)

✔ Best for full visibility
✔ Run from a pod or node

```bash
kube-hunter --pod
```

OR from node:

```bash
kube-hunter --local
```

---

## 🏗️ Recommended Approach (Your Environment)

Since you have **on-prem kubeadm cluster**:

### ✅ Step 1: Internal Scan (Node-based)

Run on master:

```bash
kube-hunter --local > master-hunter-report.txt
```

Run on worker:

```bash
kube-hunter --local > worker-hunter-report.txt
```

---

### ✅ Step 2: External Scan (Simulate attacker)

From jump host or outside network:

```bash
kube-hunter --remote <kubeapi-ip> --active > external-hunter-report.txt
```

---

## 📊 Understanding Output

### Example Findings

```
[+] Kubernetes Dashboard is accessible
[!] Kubelet allows unauthenticated access
[+] API Server is exposed
```

---

### Severity Levels

| Level    | Meaning                   |
| -------- | ------------------------- |
| Low      | Informational             |
| Medium   | Potential risk            |
| High     | Exploitable issue         |
| Critical | Immediate action required |

---

## 🔍 Common Findings & Meaning

### 🔴 Critical

* Open kubelet without auth
* Anonymous API access

### 🟡 Medium

* Dashboard exposed
* Weak TLS config

### 🟢 Low

* Version disclosure

---

## 📁 Collect Reports

```bash
scp root@master:/root/master-hunter-report.txt ./
scp root@worker:/root/worker-hunter-report.txt ./
```

---

## 🔁 Re-run After Fix

```bash
kube-hunter --local
```

---

## ⚠️ Safety Considerations

* Active mode may trigger alerts (IDS/SOC)
* Avoid running in production without approval
* Always inform security team

---

## 🔐 Enterprise Best Practices

* Run quarterly or after major changes
* Combine with:

  * kube-bench (compliance)
  * Trivy (image scan)
  * Falco (runtime detection)

---

## 🛠️ Common Findings → Exact Remediation Mapping

### 1) Anonymous access to API server

#### Finding

* API server allows anonymous requests

#### Why it is risky

* Unauthenticated users may reach discovery or other API paths
* In weak environments this can become a foothold for attackers

#### What to check

On control-plane node:

```bash
ps -ef | grep kube-apiserver | grep anonymous-auth
```

#### Secure setting

Ensure kube-apiserver runs with:

```bash
--anonymous-auth=false
```

#### Where to fix

Usually in:

```bash
/etc/kubernetes/manifests/kube-apiserver.yaml
```

Then verify:

```bash
grep anonymous-auth /etc/kubernetes/manifests/kube-apiserver.yaml
```

---

### 2) Unauthenticated kubelet access

#### Finding

* kubelet read-only or anonymous endpoints exposed

#### Why it is risky

* An attacker may enumerate pods, logs, node details, or abuse kubelet APIs

#### What to check

On each node:

```bash
ps -ef | grep kubelet
```

Also inspect kubelet config:

```bash
grep -Ei 'authentication|authorization|readOnlyPort' /var/lib/kubelet/config.yaml
```

#### Secure settings

Set:

```yaml
authentication:
  anonymous:
    enabled: false
authorization:
  mode: Webhook
readOnlyPort: 0
```

#### Where to fix

Usually in:

```bash
/var/lib/kubelet/config.yaml
```

Restart kubelet after approved change:

```bash
systemctl restart kubelet
systemctl status kubelet --no-pager
```

---

### 3) Exposed Kubernetes Dashboard

#### Finding

* Dashboard reachable from network

#### Why it is risky

* If improperly exposed, it becomes a high-value UI target

#### What to check

```bash
kubectl get svc -A | grep -i dashboard
kubectl get ingress -A | grep -i dashboard
```

#### Secure approach

* Do not expose publicly
* Restrict via internal network only
* Protect with strong authentication
* Prefer temporary access through port-forward/VPN

#### Quick validation

```bash
kubectl -n kubernetes-dashboard get svc
```

If you find `LoadBalancer`, `NodePort`, or public ingress exposure, review immediately.

---

### 4) API server exposed to untrusted networks

#### Finding

* API server port 6443 reachable more broadly than intended

#### Why it is risky

* Expands external attack surface

#### What to check

From approved test host:

```bash
nc -zv <control-plane-ip> 6443
```

On node firewall:

```bash
firewall-cmd --list-ports
firewall-cmd --list-rich-rules
```

#### Secure approach

* Allow 6443 only from:

  * admin jump hosts
  * worker node subnet
  * approved automation systems
* Block general user and internet-origin traffic

#### Example firewalld concept

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.2.0/24" port protocol="tcp" port="6443" accept'
firewall-cmd --reload
```

Adjust CIDR to your real trusted range.

---

### 5) etcd exposed or weakly protected

#### Finding

* etcd client or peer ports reachable unexpectedly

#### Why it is risky

* etcd contains highly sensitive cluster state and secrets metadata

#### What to check

On control-plane node:

```bash
ss -nltp | egrep '2379|2380'
```

#### Secure approach

* Bind etcd only to intended interfaces
* Restrict 2379/2380 to control-plane members only
* Use TLS client/server authentication

#### Where to review

```bash
/etc/kubernetes/manifests/etcd.yaml
```

Check for proper cert flags and listen/advertise addresses.

---

### 6) Insecure kubelet port 10255 open

#### Finding

* Read-only kubelet port enabled

#### Why it is risky

* Historically exposes node and pod information without strong protection

#### What to check

```bash
ss -nltp | grep 10255
```

#### Secure setting

Ensure:

```yaml
readOnlyPort: 0
```

Then restart kubelet and revalidate.

---

### 7) Kubernetes version disclosure

#### Finding

* Version/banner information easily discoverable

#### Why it matters

* Helps attacker map exploits faster

#### Practical remediation

* Keep cluster patched
* Limit exposure of API, kubelet, dashboard, ingress admin UIs
* Reduce unnecessary externally reachable control-plane services

---

## 🤖 Automation Script (Run from Each Node or Through SSH Fan-out)

### Local simple script

```bash
cat >/usr/local/bin/run-kube-hunter.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

OUT_DIR=/var/log/kube-hunter
mkdir -p "$OUT_DIR"
TS=$(date +%F-%H%M%S)
HOST=$(hostname -s)

kube-hunter --local | tee "$OUT_DIR/${HOST}-kube-hunter-${TS}.txt"
EOF

chmod +x /usr/local/bin/run-kube-hunter.sh
```

Run:

```bash
/usr/local/bin/run-kube-hunter.sh
```

---

### SSH fan-out example from jump host

```bash
for node in master01 worker01 worker02; do
  ssh root@${node} 'kube-hunter --local' > ${node}-kube-hunter.txt
done
```

This is simple and effective for internal node-based checks.

---

## 🧪 Jenkins / CI Integration Idea

### Goal

Use kube-hunter as a controlled security validation stage after infrastructure change windows.

### Example stage

```groovy
stage('Kube Hunter Scan') {
  steps {
    sh '''
      kube-hunter --remote 10.210.2.117 --active > kube-hunter-report.txt || true
    '''
    archiveArtifacts artifacts: 'kube-hunter-report.txt', fingerprint: true
  }
}
```

### Important note

* Do not run aggressive active scans casually in production pipelines
* Prefer scheduled or manually approved jobs
* Inform SOC / security operations before active probing

---

## 📎 Verification Checklist

✔ kube-hunter installed
✔ Internal scan executed
✔ External scan executed
✔ Reports collected
✔ Critical issues addressed

---

## 🎯 Golden Commands

### Internal Scan

```bash
kube-hunter --local
```

### External Active Scan

```bash
kube-hunter --remote <IP> --active
```

---

## 🚀 Outcome

You will achieve:

* Visibility from attacker perspective
* Exposure detection
* Network security validation

---

## 📌 End of Runbook
