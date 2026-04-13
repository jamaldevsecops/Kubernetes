# 🚀 Kubernetes Demo Deployment Guide

### 🛍️ Online Boutique (Microservices Demo) with Traefik Ingress

---

## 📌 Purpose of the Deployment

This deployment is intended for:

* 🧪 **Testing Kubernetes networking**
* 🔐 **Validating NetworkPolicy behavior**
* 🌐 **Practicing Ingress (Traefik) routing**
* 🔍 **Observing service-to-service communication**
* ⚙️ **DevOps lab / interview preparation**

The application is a **microservices-based e-commerce demo** consisting of:

* Frontend (UI)
* Multiple backend services
* Redis (cart storage)

---
## 🔗 References
- GitHub Repo: https://github.com/GoogleCloudPlatform/microservices-demo  

## 🧱 Architecture Overview
![Online Boutique Architecture](https://github.com/GoogleCloudPlatform/microservices-demo/blob/main/docs/img/architecture-diagram.png)

---

## 📦 Deployment Steps

### 🔹 1. Clone the Repository

```bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
```

---

### 🔹 2. Create Namespace

```bash
kubectl create namespace boutique
```

---

### 🔹 3. Deploy Application

```bash
kubectl apply -n boutique -f ./release/kubernetes-manifests.yaml
```

---

### 🔹 4. Verify Deployment

```bash
kubectl get pods -n boutique
```
Sample Output: 
```bash
NAME                                     READY   STATUS    RESTARTS      AGE
adservice-848c5d6f88-mvnq6               1/1     Running   0             18m
cartservice-59d44fb67-4d2h2              1/1     Running   0             18m
checkoutservice-54475449f4-gp8xb         1/1     Running   0             18m
currencyservice-6bbd8c95f4-l6k5n         1/1     Running   0             18m
emailservice-68dd7ccf64-kbtjd            1/1     Running   3 (17m ago)   18m
frontend-6b8fcb997-bsp4p                 1/1     Running   0             18m
loadgenerator-8599589654-86l87           1/1     Running   0             18m
paymentservice-cc458477b-47hvs           1/1     Running   0             18m
productcatalogservice-7d7957447b-8q6j5   1/1     Running   0             18m
recommendationservice-84d6f4488d-sxxbx   1/1     Running   0             18m
redis-cart-c4fc658fb-xzz7l               1/1     Running   0             18m
shippingservice-69f6756b4d-jn2sk         1/1     Running   0             18m
```
```bash
kubectl get svc -n boutique
```
```bash
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
adservice               ClusterIP      10.110.152.199   <none>           9555/TCP       2m2s
cartservice             ClusterIP      10.103.253.152   <none>           7070/TCP       2m3s
checkoutservice         ClusterIP      10.102.235.203   <none>           5050/TCP       2m3s
currencyservice         ClusterIP      10.101.148.3     <none>           7000/TCP       2m3s
emailservice            ClusterIP      10.97.58.162     <none>           5000/TCP       2m3s
frontend                ClusterIP      10.105.45.4      <none>           80/TCP         2m2s
frontend-external       LoadBalancer   10.103.26.90     192.168.20.161   80:32541/TCP   2m2s
paymentservice          ClusterIP      10.110.171.61    <none>           50051/TCP      2m3s
productcatalogservice   ClusterIP      10.96.158.23     <none>           3550/TCP       2m3s
recommendationservice   ClusterIP      10.109.49.150    <none>           8080/TCP       2m2s
redis-cart              ClusterIP      10.101.182.122   <none>           6379/TCP       2m3s
shippingservice         ClusterIP      10.105.231.250   <none>           50051/TCP      2m3s
```

✅ Ensure all pods are **Running**

---

## 🌐 Accessing the Frontend

The application provides multiple access methods:

---

# 🔹 Option 1: Access via LoadBalancer (MetalLB)

If you are using **MetalLB**, you will see:

```bash
kubectl get svc -n boutique
```

Example:

```
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
frontend-external       LoadBalancer   10.103.26.90     192.168.20.161   80:32541/TCP   2m2s
```

### ✅ Access URL:

```
http://192.168.20.161
```

---

### ⚠️ Notes

* Works only if MetalLB IP is reachable from your network
* Ensure firewall allows access

---

# 🔹 Option 2: Access via Traefik IngressRoute (Recommended)

## 🧠 Why use Ingress?

* Clean architecture
* Single entry point
* TLS support
* Security control
* Production-ready pattern

---

## 📌 Internal Service Used

* `frontend` → ClusterIP (used by Traefik)
* `frontend-external` → optional (can be removed)

---

## 🛠️ Create IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: boutique-frontend
  namespace: boutique
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`boutique.apsis.localnet`)
      kind: Rule
      services:
        - name: frontend
          port: 80
```

---

### 🔹 Apply

```bash
kubectl apply -f ingressroute.yaml
```

---

## 🌍 DNS / Hosts Setup

Add entry to your local machine:

```bash
sudo nano /etc/hosts
```

```
192.168.20.163 boutique.apsis.localnet
```
![Web UI](<img width="1661" height="682" alt="image" src="https://github.com/user-attachments/assets/1e283c1e-0601-4c72-bb24-dbbb71f2f09d" />)

---

## ✅ Access

```
http://boutique.apsis.localnet
```

---

## 🔐 (Optional) Enable HTTPS

```yaml
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`boutique.apsis.localnet`)
      kind: Rule
      services:
        - name: frontend
          port: 80
  tls: {}
```

---

## 🔁 HTTP → HTTPS Redirect (Optional)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: boutique
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

---

## 🧹 Cleanup (Optional)

Remove LoadBalancer if using Traefik:

```bash
kubectl delete svc frontend-external -n boutique
```

---

## 🔍 Verification

```bash
kubectl get ingressroute -n boutique
kubectl describe ingressroute boutique-frontend -n boutique
kubectl logs -n <traefik-namespace> deploy/<traefik>
```

---

## 🧪 Recommended Testing Scenarios

* ❌ Block frontend → backend (NetworkPolicy)
* ❌ Block backend → Redis
* ❌ Block DNS (port 53)
* 🔍 Observe failures via UI

---

## ⚠️ Common Issues

| Issue               | Cause                      |
| ------------------- | -------------------------- |
| Cannot access LB IP | MetalLB routing / firewall |
| Ingress not working | DNS / entryPoint mismatch  |
| Service unreachable | NetworkPolicy blocking     |
| DNS failure         | Missing egress rule        |

---

## 🎯 Best Practice (Production Style)

* Use **ClusterIP services internally**
* Use **Ingress (Traefik) externally**
* Apply **default deny NetworkPolicy**
* Allow only required traffic
* Enable TLS + security middleware

---

## ✅ Summary

| Method       | Use Case                 |
| ------------ | ------------------------ |
| LoadBalancer | Quick testing            |
| NodePort     | Fallback                 |
| Port-forward | Debugging                |
| IngressRoute | Production / Clean setup |

---

## 🚀 Next Steps

* 🔐 Apply NetworkPolicy
* 📊 Monitor traffic
* 🧰 Add observability (Prometheus/Grafana)
* 🔎 Use tools like Hubble / Calico flow logs

---

**Happy Testing! 🎉**
