# 🚨 Kubernetes Network Policy Troubleshooting

## ❓ What is a Network Policy?
A **NetworkPolicy** in Kubernetes is used to control traffic between pods and/or from/to the outside world based on rules.  
It operates at OSI Layer 3/4 and requires a CNI plugin like **Calico**, **Cilium**, or **Weave**.

> 🔥 Misconfiguration can silently break pod-to-pod or pod-to-service communication — especially for databases and APIs.

---

## 🧩 Real-World Scenario

### ❗ Issue
After deploying a microservice app in production, the frontend or service layer can't reach the backend API or database (PostgreSQL/MySQL).

### 🔍 Root Cause
A **NetworkPolicy** on the `db` namespace only allowed traffic from pods with label `app=db-admin`.  
Your microservice pod lacked this label → connection was **denied silently**.

---

## 🛠️ Step-by-Step Troubleshooting

| Step | Action |
|------|--------|
| 1️⃣ | List Network Policies:<br>`kubectl get networkpolicy -n <namespace>` |
| 2️⃣ | Describe the Policy:<br>`kubectl describe networkpolicy <name> -n <namespace>` |
| 3️⃣ | Check Pod Labels:<br>`kubectl get pods -n <namespace> --show-labels` |
| 4️⃣ | Run Debug Pod:<br>`kubectl run tmp-shell --rm -i -t --image=busybox -- /bin/sh`<br>Inside pod: `nc -zv <db-service> 5432` |
| 5️⃣ | Temporarily delete the policy:<br>`kubectl delete networkpolicy <name> -n <namespace>`<br>🔁 If issue resolves → root cause confirmed. |

---

## ✅ Fix

Update the NetworkPolicy to include the correct pod label:

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: your-app
```

## 🔒 Best Practices for Securing Database Access in K8s

| Practice               | Recommendation                                                                 |
|------------------------|---------------------------------------------------------------------------------|
| NetworkPolicy          | Limit DB access to necessary pods only                                          |
| TLS Encryption         | Use SSL/TLS for DB connections (`sslmode=require` for PostgreSQL)              |
| Secrets Management     | Store credentials securely in Kubernetes Secrets                                |
| RBAC                   | Restrict Secret access using scoped service accounts                            |
| Private Service        | Expose DB only internally via ClusterIP (no external exposure)                  |
| IRSA (on EKS)          | Use IAM Roles for Service Accounts for AWS RDS integration                      |
| PodSecurity/AppArmor   | Apply PodSecurity policies like AppArmor or Seccomp                             |
| Monitoring & Backup    | Monitor DB access and schedule backups                                          |
| Connection Pooler      | Use PgBouncer or similar to manage DB load from microservices                   |

# 🔬 Lab: Simulating NetworkPolicy Troubleshooting in Kubernetes

## 🎯 Goal

Simulate a broken `app → db` connection due to a restrictive `NetworkPolicy`, then troubleshoot and fix it by updating pod labels.

---

## 🧱 Lab Setup

- **Namespace**: `netpol-demo`
- **Pods**:
  - `app`: Busybox pod simulating a microservice
  - `db`: Nginx pod simulating a database
- **Policy**: Deny all ingress to `db`, allow only from pods with label `role=admin`

---

## ✅ Step 1: Create Namespace

```bash
kubectl create namespace netpol-demo
```
## ✅ Step 2: Deploy DB Pod
```yaml
cat <<EOF | kubectl apply -n netpol-demo -f -
apiVersion: v1
kind: Pod
metadata:
  name: db
  labels:
    role: db
spec:
  containers:
  - name: db
    image: nginx
    ports:
    - containerPort: 80
EOF
```
## ✅ Step 3: Deploy App Pod
```yaml
cat <<EOF | kubectl apply -n netpol-demo -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    role: client
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
EOF
```
## ✅ Step 4: Test Initial Connectivity (Should Work)
```bash
kubectl exec -n netpol-demo app -- wget -qO- http://db
```
✅ You should see the Nginx welcome page.
## ✅ Step 5: Apply Restrictive NetworkPolicy
```yaml
cat <<EOF | kubectl apply -n netpol-demo -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-except-admin
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: admin
    ports:
    - protocol: TCP
      port: 80
  policyTypes:
  - Ingress
EOF
```
🔒 Now, only pods with label role=admin can access the db pod.
## ❌ Step 6: Retest (Should Fail)
```bash
kubectl exec -n netpol-demo app -- wget -qO- http://db
```
❌ This should hang or fail — access is blocked by the NetworkPolicy.
## 🔍 Step 7: Troubleshoot
### ✅ Check NetworkPolicy
```bash
kubectl describe networkpolicy deny-all-except-admin -n netpol-demo
```
### ✅ Check Pod Labels
```bash
kubectl get pods -n netpol-demo --show-labels
```
⚠️ You'll see that the app pod has role=client, which is not allowed by the policy.
## ✅ Step 8: Fix — Patch App Pod Label
```bash
kubectl label pod app role=admin --overwrite -n netpol-demo
```
## ✅ Step 9: Retest
```bash
kubectl exec -n netpol-demo app -- wget -qO- http://db
``
✅ Now the app can reach the DB pod — issue resolved!
## 🧹 Cleanup
```bash
kubectl delete namespace netpol-demo
```


