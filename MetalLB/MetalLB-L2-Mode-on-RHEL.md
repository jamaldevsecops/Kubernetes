# 🚀 MetalLB L2 Mode Installation Runbook

**Environment:** RHEL 8 + Kubernetes v1.35.x + CRI-O + Calico
**Version:** MetalLB v0.15.3
**Mode:** Layer 2 (L2)
**IP Pool:** 10.210.2.122–10.210.2.124

---

# 📌 1. Objective

Deploy MetalLB in **Layer 2 mode** to provide `LoadBalancer` service support for on-prem Kubernetes cluster.

---

# 🧱 2. Prerequisites

## ✅ Cluster Requirements

* Kubernetes cluster (kubeadm-based)
* All nodes in `Ready` state
* CNI (Calico) installed and healthy
* CRI-O runtime functional

## ✅ Network Requirements

* Nodes in subnet: `10.210.2.0/24`
* Reserved IP range:

  * `10.210.2.122–124` (must NOT be used elsewhere)
* Same L2 network (no routing between nodes and clients)

## ⚠️ Important Checks

```bash
ping 10.210.2.122
arp -an | grep 10.210.2.122
```

✔ Ensure IPs are:

* Not assigned
* Not in DHCP pool
* Not used by F5 / gateway / servers

---

# 📦 3. Install MetalLB (Official Manifest)

## ▶️ Apply official manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

---

## ▶️ Verify installation

```bash
kubectl get pods -n metallb-system
```

### ✅ Expected Output

* controller pod (1)
* speaker pods (one per node)

---

# 🔐 4. Create IP Pool & L2 Advertisement

## ▶️ Create configuration file

```bash
cat > metallb-l2-config.yaml <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.210.2.122-10.210.2.124

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: production-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
EOF
```

---

## ▶️ Apply configuration

```bash
kubectl apply -f metallb-l2-config.yaml
```

---

## ▶️ Verify resources

```bash
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```

---

# 🧪 5. Functional Test (LoadBalancer)

## ▶️ Deploy test application

```bash
kubectl create deployment nginx-test --image=nginx
kubectl expose deployment nginx-test \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer
```

---

## ▶️ Verify service

```bash
kubectl get svc -o wide
```

### ✅ Expected Output

```
EXTERNAL-IP: 10.210.2.122 (or .123 / .124)
```

---

# 🌐 6. Connectivity Validation

## ▶️ From same network machine

```bash
ping 10.210.2.122
curl http://10.210.2.122
arp -an | grep 10.210.2.122
```

---

## 🧠 Expected Behavior

* One node (speaker) will **announce IP via ARP**
* Traffic will flow:

```
Client → VIP → Node → Pod
```

---

# 🔍 7. Troubleshooting Guide

## ❌ EXTERNAL-IP Pending

### Check:

```bash
kubectl describe svc nginx-test
kubectl get events -A | grep metallb
```

---

## ❌ Speaker Issues

```bash
kubectl logs -n metallb-system -l component=speaker
```

---

## ❌ Controller Issues

```bash
kubectl logs -n metallb-system deploy/controller
```

---

## ❌ Network Issues

Verify:

* Same subnet
* No firewall blocking ARP
* No IP conflict

---

# 🔐 8. Production Hardening Checklist

## ✅ Network

* Reserve IP range in network team documentation
* Exclude from DHCP scope

## ✅ Security

* Restrict access via NetworkPolicy (Calico)
* Limit LoadBalancer exposure (internal vs external)

## ✅ Observability

* Monitor:

  * speaker pod restarts
  * ARP announcements
  * service availability

## ✅ High Availability

* Ensure:

  * multiple worker nodes
  * speaker runs on all nodes

---

# 🧹 9. Cleanup (Test Resources)

```bash
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
```

---

# 📊 10. Architecture Overview

```
                +----------------------+
                |     Client Network   |
                |   10.210.2.0/24      |
                +----------+-----------+
                           |
                     (ARP Request)
                           |
                    +------v------+
                    | MetalLB VIP |
                    |10.210.2.122 |
                    +------+------+
                           |
                +----------v----------+
                |  Kubernetes Node    |
                | (Speaker Active)    |
                +----------+----------+
                           |
                    +------v------+
                    |   Service   |
                    +------v------+
                    |     Pod     |
                    +-------------+
```

---

# 🧠 11. Key Notes (Enterprise Insight)

* L2 mode = simple, no router config
* Not ideal for large-scale (use BGP for scaling)
* Single node handles traffic per service (no ECMP)
* Failover is fast but not instant (~seconds)

---

# 🏁 Final Status

✔ MetalLB Installed
✔ IP Pool Configured
✔ LoadBalancer Working
✔ Cluster Ready for Ingress / Production Traffic

---
