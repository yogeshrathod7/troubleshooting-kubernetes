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

