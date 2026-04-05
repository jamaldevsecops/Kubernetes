# 🚀 MetalLB L2 Mode Installation Runbook

**Environment:** RHEL 8 | Kubernetes v1.35.x | CRI-O | Calico
**Version:** MetalLB v0.15.3
**Mode:** Layer 2 (L2)
**IP Pool:** 10.210.2.122–10.210.2.124

---

# 🎯 1. Objective

Deploy MetalLB in **Layer 2 mode** to enable `LoadBalancer` services in an on-prem Kubernetes cluster with **production-grade reliability and security**.

---

# 🧱 2. Prerequisites

## ✅ Cluster Requirements

* Kubernetes cluster (kubeadm-based)
* All nodes in `Ready` state
* Calico CNI installed and healthy
* CRI-O runtime operational

## 🌐 Network Requirements

* Node subnet: `10.210.2.0/24`
* Reserved IP range:
  `10.210.2.122 – 10.210.2.124`
* Same Layer 2 network across nodes and clients

---

## 🔍 2.1 Validate Cluster Health

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

✔ All nodes must be `Ready`
✔ All system pods must be `Running`

---

## 🔎 2.2 Validate IP Availability

```bash
ping 10.210.2.122
arp -an | grep 10.210.2.122
```

✔ No response indicates IP is available

---

# 🔥 3. Firewall Configuration (firewalld)

## ⚙️ Verify firewall status

```bash
firewall-cmd --state
firewall-cmd --list-all
```

---

## 🛡️ Allow required Kubernetes & networking traffic on Master Node

```bash
# 🔐 Kubernetes API Server (worker → master communication)
firewall-cmd --permanent --add-port=6443/tcp

# 🧠 etcd cluster communication (control-plane internal)
firewall-cmd --permanent --add-port=2379-2380/tcp

# 🤖 kubelet API (metrics, exec, logs)
firewall-cmd --permanent --add-port=10250/tcp

# 🎛️ Controller Manager
firewall-cmd --permanent --add-port=10257/tcp

# 🗂️ Scheduler
firewall-cmd --permanent --add-port=10259/tcp

# 🔁 kube-proxy (service routing health checks)
firewall-cmd --permanent --add-port=10256/tcp

# 🌐 Calico BGP (only if BGP mode is used; safe to keep open)
firewall-cmd --permanent --add-port=179/tcp

# 🌉 Calico VXLAN overlay (CRITICAL for pod networking)
firewall-cmd --permanent --add-port=4789/udp

# 📦 Calico IP-in-IP (if enabled)
firewall-cmd --permanent --add-protocol=ipip

# 🧩 Allow POD network (cluster internal traffic)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address=10.244.0.0/16 accept'

# 🧩 Allow SERVICE network (ClusterIP range)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address=10.96.0.0/12 accept'

# 🔄 Enable NAT / forwarding (required for pod routing & service access)
firewall-cmd --permanent --add-masquerade

# 🔁 Apply changes
firewall-cmd --reload
```

## 🛡️ Allow required Kubernetes & networking traffic on Worker Nodes

```bash
# 🤖 kubelet API (master → worker communication)
firewall-cmd --permanent --add-port=10250/tcp

# 🔁 kube-proxy (service routing)
firewall-cmd --permanent --add-port=10256/tcp

# 🌐 Calico BGP (only required if BGP mode is used)
firewall-cmd --permanent --add-port=179/tcp

# 🌉 Calico VXLAN overlay (CRITICAL for pod networking)
firewall-cmd --permanent --add-port=4789/udp

# 📦 Calico IP-in-IP (if enabled)
firewall-cmd --permanent --add-protocol=ipip

# 🌍 NodePort services (external access to services)
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --permanent --add-port=30000-32767/udp

# 🧩 Allow POD network (cluster internal traffic)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address=10.244.0.0/16 accept'

# 🧩 Allow SERVICE network (ClusterIP range)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address=10.96.0.0/12 accept'

# 🔄 Enable NAT / forwarding (required for pod routing & service access)
firewall-cmd --permanent --add-masquerade

# 🔁 Apply changes
firewall-cmd --reload
```
---

# 🌐 4. Proxy Configuration (if applicable)

## ⚙️ Configure CRI-O proxy

```bash
mkdir -p /etc/systemd/system/crio.service.d/

cat <<EOF > /etc/systemd/system/crio.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://<proxy>:<port>"
Environment="HTTPS_PROXY=http://<proxy>:<port>"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,10.96.0.0/12,10.244.0.0/16,.svc,.cluster.local"
EOF

systemctl daemon-reexec
systemctl restart crio
```

---

# 📦 5. Install MetalLB

## ▶️ Apply official manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

---

## ⏳ Wait for readiness

```bash
kubectl wait -n metallb-system \
  --for=condition=Available deployment/controller \
  --timeout=180s
```

---

# 🔍 6. Verify Installation

## ▶️ Check pods

```bash
kubectl get pods -n metallb-system
```

✔ controller → Running  
✔ speaker → Running on all nodes

---

## ▶️ Check webhook service

```bash
kubectl get svc metallb-webhook-service -n metallb-system
```

---

## ▶️ Validate backend using EndpointSlice (modern method)

```bash
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service
```

✔ Must show at least one endpoint

---

# 🔐 7. Configure IP Pool & L2 Advertisement

## ▶️ Create configuration

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

# 🧪 8. Functional Testing

## ▶️ Deploy test workload

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

✔ Expected:

```
EXTERNAL-IP = 10.210.2.122-124
```

---

# 🌐 9. Connectivity Validation

```bash
curl http://10.210.2.122
arp -an | grep 10.210.2.122
```

---

# 🧠 10. Architecture Overview

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

# 🔐 11. Production Best Practices

## 🛡️ Security

* Apply Calico NetworkPolicy
* Restrict unnecessary exposure

## 📊 Monitoring

* Monitor MetalLB speaker pods
* Alert on restarts or failures

## 🌐 Network Governance

* Reserve IP pool in network documentation
* Exclude from DHCP scope

## ⚙️ Stability

* Ensure multi-node deployment
* Validate failover behavior

---

# 🧹 12. Cleanup (Test Resources)

```bash
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
```

---

# 🏁 Final Status

✅ MetalLB Installed
✅ L2 Mode Configured
✅ LoadBalancer Functional
✅ Production Ready

---
