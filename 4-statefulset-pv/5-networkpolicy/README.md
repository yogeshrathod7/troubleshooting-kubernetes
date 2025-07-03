# 🚨 Kubernetes Network Policy Troubleshooting

## ❓ What is a NetworkPolicy?

A **NetworkPolicy** in Kubernetes restricts traffic between pods and external sources based on label-based rules. It operates at the **network layer (L3/L4)** and requires a CNI plugin like **Calico**, **Cilium**, or **Weave**.

> ⚠️ Misconfigured policies can silently break communication between services.

---

## 🔍 Real-Time Issue: App Can't Connect to DB

### 📌 Scenario
- Microservice app fails to connect to database
- A NetworkPolicy allows traffic **only** from `app=db-admin`
- App pod has the wrong label → connection denied (timeout or refused)

---

## 🛠️ Quick Troubleshooting Steps

```bash
# 1. Check policies
kubectl get networkpolicy -n <namespace>

# 2. Inspect policy rules
kubectl describe networkpolicy <name> -n <namespace>

# 3. Verify pod labels
kubectl get pods -n <namespace> --show-labels

# 4. Debug from inside the cluster
kubectl run tmp-shell -it --rm --image=busybox -- sh
nc -zv <db-service> 5432  # or your DB port

# 5. Temporarily remove policy (optional)
kubectl delete networkpolicy <name> -n <namespace>
## 🔒 Best Practices for Securing Databases in Kubernetes
*Practice*	                    *Recommendation*
✅ NetworkPolicy	      Restrict DB access to necessary pods only
🔐 Secrets	Store       credentials securely
🔑 RBAC	Limit Secret    access using service accounts
🔒 TLS	                Use encrypted DB connections (e.g., sslmode=require)
🌐 ClusterIP	          Avoid exposing DB externally
🛡️ PodSecurity	        Use AppArmor, Seccomp, PSP
🔁 Connection Pooler	  Use PgBouncer for efficient DB access
#🔬 Lab: Simulate and Fix NetworkPolicy Issue
##🧱 Setup
Namespace: netpol-demo

Pods:

  app → busybox (client)

  db → nginx (mock DB)

Policy: only allow ingress to db from pods labeled role=admin
# 1. Create namespace
kubectl create ns netpol-demo

# 2. Deploy db pod
kubectl run db -n netpol-demo --image=nginx --labels="role=db" --port=80 --restart=Never

# 3. Deploy app pod
kubectl run app -n netpol-demo --image=busybox --labels="role=client" --command -- sleep 3600

# 4. Test connectivity
kubectl exec -n netpol-demo app -- wget -qO- http://db

# 5. Apply restrictive policy
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

# 6. Retest — expected to fail
kubectl exec -n netpol-demo app -- wget -qO- http://db

# 7. Fix label on app pod
kubectl label pod app role=admin --overwrite -n netpol-demo

# 8. Retest — should succeed
kubectl exec -n netpol-demo app -- wget -qO- http://db

# 9. Cleanup
kubectl delete ns netpol-demo
