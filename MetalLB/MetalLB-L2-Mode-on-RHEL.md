# MetalLB L2 Mode Runbook (Updated for EndpointSlice Warning)

**Environment**
- OS: RHEL 8
- Kubernetes: v1.35.x (kubeadm)
- Runtime: CRI-O
- CNI: Calico
- Firewall: firewalld enabled
- Proxy: enabled for outbound internet access
- MetalLB: v0.15.3
- Mode: Layer 2 (L2)
- IP Pool: `10.210.2.122-10.210.2.124`

---

## 1. Purpose

This runbook installs MetalLB in **L2 mode** using the official **v0.15.3** manifest and uses **EndpointSlice-aware validation commands** to avoid the deprecated `Endpoints` warning introduced in Kubernetes v1.33+. As of Kubernetes v1.33, the old `Endpoints` API is deprecated and users should move scripts and operational checks to `EndpointSlice` (`discovery.k8s.io/v1`).

---

## 2. Important Note About Your Warning

When you ran:

```bash
kubectl get svc,endpoints -n metallb-system
```

you received:

```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
```

This is **not a failure**. It is only a **deprecation warning** because `kubectl` is reading the legacy `Endpoints` resource. The correct production approach is:

- keep using `Service` objects as usual,
- stop using `kubectl get endpoints ...` in checks and scripts,
- use `EndpointSlice` instead.

---

## 3. Production-Safe Replacement Commands

### Old command (deprecated)

```bash
kubectl get svc,endpoints -n metallb-system
```

### New commands (recommended)

```bash
kubectl get svc -n metallb-system
kubectl get endpointslices.discovery.k8s.io -n metallb-system
```

### Best focused check for MetalLB webhook backend

```bash
kubectl get svc metallb-webhook-service -n metallb-system
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service
```

### Detailed YAML view

```bash
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service -o yaml
```

---

## 4. Pre-Checks

### 4.1 Cluster health

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

Expected:
- all nodes `Ready`
- no critical pods in `CrashLoopBackOff`

### 4.2 Check firewall state on each node

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all
```

### 4.3 Check proxy-related NO_PROXY coverage

Ensure the following are excluded from proxying in node/container runtime configuration:

```text
localhost,127.0.0.1,10.0.0.0/8,10.96.0.0/12,10.244.0.0/16,.svc,.cluster.local
```

This is important because cluster-internal traffic such as API server to webhook service must stay internal.

### 4.4 Confirm IP pool is free

```bash
ping -c 2 10.210.2.122
ping -c 2 10.210.2.123
ping -c 2 10.210.2.124
arp -an | egrep '10.210.2.122|10.210.2.123|10.210.2.124'
```

Expected:
- no active host response
- no conflicting ARP entry from another device

---

## 5. Firewall Guidance

If `firewalld` is enabled, make sure Kubernetes and Calico traffic is not blocked.

### Minimum common Kubernetes ports

Run as needed based on your node role and policy:

```bash
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10256/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --permanent --add-port=30000-32767/udp
```

### Calico-related allowances often needed

```bash
firewall-cmd --permanent --add-port=179/tcp
firewall-cmd --permanent --add-protocol=ipip
```

### Allow pod and service CIDRs internally

```bash
firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="10.244.0.0/16" accept'
firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="10.96.0.0/12" accept'
firewall-cmd --reload
```

> Adjust these if your actual Service CIDR differs from `10.96.0.0/12`.

---

## 6. Install MetalLB v0.15.3

Apply the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

Wait for readiness:

```bash
kubectl wait -n metallb-system --for=condition=Available deployment/controller --timeout=180s
kubectl get pods -n metallb-system -o wide
```

Expected:
- `controller` deployment available
- `speaker` daemonset pods running on nodes

---

## 7. Validate Webhook Using EndpointSlice-Aware Checks

### Check service

```bash
kubectl get svc metallb-webhook-service -n metallb-system
```

### Check EndpointSlice instead of Endpoints

```bash
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service
```

### Example interpretation

If you see one or more EndpointSlice objects with backend addresses and port `9443`, the webhook backend exists and the service has usable endpoints.

### Optional detailed description

```bash
kubectl describe svc metallb-webhook-service -n metallb-system
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service -o yaml
```

---

## 8. Create MetalLB L2 Configuration

Create the configuration file:

```bash
cat > metallb-l2-config.yaml <<'EOFCONF'
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
EOFCONF
```

Apply it:

```bash
kubectl apply -f metallb-l2-config.yaml
```

Verify:

```bash
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```

---

## 9. Functional Test

Deploy a test service:

```bash
kubectl create deployment nginx-test --image=nginx
kubectl expose deployment nginx-test --port=80 --target-port=80 --type=LoadBalancer
kubectl get svc nginx-test -o wide
```

Expected:
- external IP assigned from:
  - `10.210.2.122`
  - `10.210.2.123`
  - `10.210.2.124`

Validate from another host on the same network:

```bash
curl http://10.210.2.122
arp -an | grep 10.210.2.122
```

---

## 10. Webhook Troubleshooting

If applying `metallb-l2-config.yaml` fails with webhook timeout:

### 10.1 Check MetalLB pods

```bash
kubectl get pods -n metallb-system -o wide
```

### 10.2 Check controller logs

```bash
kubectl logs -n metallb-system deploy/controller --tail=200
```

### 10.3 Check service and EndpointSlice

```bash
kubectl get svc metallb-webhook-service -n metallb-system
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service
```

### 10.4 Check kube-proxy and Calico

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100
kubectl logs -n kube-system -l k8s-app=calico-node --tail=100
```

### 10.5 Check firewall again

```bash
firewall-cmd --list-all
```

Most likely causes:
- MetalLB controller not ready
- webhook backend not published into EndpointSlice
- firewall blocking service/pod traffic
- NO_PROXY missing cluster-internal ranges/domains

---

## 11. Updated Operational Commands

Use these in your day-to-day checks to avoid the deprecation warning.

### MetalLB namespace health

```bash
kubectl get pods -n metallb-system -o wide
kubectl get svc -n metallb-system
kubectl get endpointslices.discovery.k8s.io -n metallb-system
```

### Webhook backend health

```bash
kubectl get svc metallb-webhook-service -n metallb-system
kubectl get endpointslices.discovery.k8s.io -n metallb-system \
  -l kubernetes.io/service-name=metallb-webhook-service
```

### Service exposure check

```bash
kubectl get svc -A
```

---

## 12. Cleanup Test Resources

```bash
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
```

---

## 13. Final Notes

- The warning you saw is expected on Kubernetes v1.33+ when reading `Endpoints`.
- Your output already showed the webhook backend exists; in modern checks, confirm this through `EndpointSlice` instead of `Endpoints`.
- For production scripts, monitoring, and runbooks, replace all `kubectl get endpoints ...` usage with `kubectl get endpointslices.discovery.k8s.io ...`.

