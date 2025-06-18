## 1️⃣ Scenario: Wrong Image Name or Tag

A common typo like `nginx:latets` (instead of `nginx:latest`) can prevent Kubernetes from pulling the container image.

---

## 🔧 Simulate the Issue

### Step 1: Create a broken Pod manifest

```yaml
# wrong-image.yaml
apiVersion: v1
kind: Pod
metadata:
  name: wrong-image
spec:
  containers:
  - name: app
    image: nginx:latets
```
### Step 2: Apply the manifest
kubectl apply -f wrong-image.yaml
