# Kubernetes Troubleshooting Scenarios

A quick reference for diagnosing and fixing common Kubernetes pod and workload issues in production or lab environments.

---

## 1️⃣ ImagePullBackOff

### What Is It?
Kubernetes can't pull the container image for your Pod.

### Common Causes:
- Incorrect image name or tag
- Missing or incorrect `imagePullSecrets`
- Private registry access issues

### Troubleshooting:
```bash
kubectl describe pod <pod-name>
kubectl get secret <secret-name> -n <namespace>
```
### ✅ Quick Fix
- Check image name and tag
- Create a valid secret:
  ` kubectl create secret docker-registry regcred \
  --docker-username=<user> --docker-password=<pass> \
  --docker-server=<registry> -n <namespace> `
- Add to Pod/Deployment spec:
  ```yaml
  imagePullSecrets:
  - name: regcred
  ```
## 2️⃣ CrashLoopBackOff
### What Is It?
Pod crashes repeatedly and Kubernetes keeps restarting it.
### Common Causes:
- App error or non-zero exit code
- Missing ConfigMap/Secret
- Init container failure
### Troubleshooting:
```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```
### ✅ Quick Fix
- Fix startup bug or config.
- Validate secrets, volumes, init containers.
- Add readiness/liveness probes appropriately.
## 3️⃣ Pods Not Schedulable
### What Is It?
Pods remain in Pending state due to scheduling issues.
### Common Causes:
- No available nodes with required resources
- Node selector/taints mismatch
- PersistentVolumeClaims unbound
### Troubleshooting
```bash
kubectl describe pod <pod-name>
kubectl get nodes -o wide
kubectl get pvc
```
###✅ Quick Fix
- Adjust resource requests/limits
- Remove taints or use tolerations
- Ensure matching PVC and StorageClass
## 4️⃣ StatefulSet Volume Issues (PVC Unbound)
### What Is It?
StatefulSet pods stuck due to unbound or missing PVCs.
### Common Causes
- Incorrect StorageClass
- Missing volume binding in zone
- EBS CSI driver not installed (AWS)
### Troubleshooting
``` bash
kubectl get pvc -n <namespace>
kubectl describe pvc <name>
kubectl describe statefulset <name>
```
### ✅ Quick Fix
- Install correct CSI driver (e.g., AWS EBS CSI)
- Ensure volumeBindingMode and zone match
- Use VolumeSnapshots for restore if needed

## 5️⃣ NetworkPolicy Denying Traffic
### What Is It?
Pods can't communicate due to restrictive NetworkPolicy rules.
### Common Causes:
- Pod labels not matching policy
- No ingress/egress rules defined (deny-by-default)
### Troubleshooting
```bash
kubectl describe networkpolicy <name> -n <namespace>
kubectl get pods -n <namespace> --show-labels
kubectl exec -it <pod> -- nc -zv <target> <port>
```
### ✅ Quick Fix 
-  Update NetworkPolicy to allow proper labels:
  ```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: my-app
```
## 📦 Pro Tips

| 🔧 Area     | 💡 Recommendation                                                                 |
|------------|------------------------------------------------------------------------------------|
| Logs       | Always check `kubectl logs <pod>` and `kubectl describe <pod>`                    |
| Debugging  | Use ephemeral debug pods: `kubectl run -it debug --image=busybox --rm -- /bin/sh` |
| Labels     | Double-check labels used in NetworkPolicy, Service, and Deployment selectors       |
| Volumes    | Ensure correct `StorageClass`, matching `volumeBindingMode`, and PVCs             |
| Events     | Check cluster events: `kubectl get events --sort-by='.metadata.creationTimestamp'`|



