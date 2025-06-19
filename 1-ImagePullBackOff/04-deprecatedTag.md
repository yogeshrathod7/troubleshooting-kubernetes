## 4️⃣ Scenario: Deleted or Deprecated Image Tag

A Pod may fail with `ImagePullBackOff` if the specified image tag was deleted, deprecated, or never existed.

---

## 🔧 Simulate the Issue

Create the following manifest using an invalid or outdated image tag.

File: `deleted-tag.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: deleted-tag
spec:
  containers:
  - name: app
    image: nginx:0.1  # assume this tag doesn't exist
```
Apply it:
```
kubectl apply -f deleted-tag.yaml
```
Check the Pod status:
```
kubectl describe pod deleted-tag
```
## 🔍 Sample Error Output
```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  ...                default-scheduler  Assigned to node
  Normal   Pulling    ...                kubelet            Pulling image "nginx:0.1"
  Warning  Failed     ...                kubelet            Failed to pull image "nginx:0.1": tag not found
  Warning  Failed     ...                kubelet            Error: ErrImagePull
  Normal   BackOff    ...                kubelet            Back-off pulling image
  Warning  Failed     ...                kubelet            Error: ImagePullBackOff
```
## 🛠️ Fix the Problem
### Step 1: Edit the Pod
```
kubectl edit pod deleted-tag
```
### Step 2: Change the image tag
From:
```
image: nginx:0.1
```
To:
```
image: nginx:1.25
```
You can replace 1.25 with any valid and available tag from Docker Hub - nginx.

### Step 3: Verify the Fix
```
kubectl get pods
```
Expected output:
```
NAME          READY   STATUS    RESTARTS   AGE
deleted-tag   1/1     Running   0          45s
```
### ✅ Summary
| Problem                 | Root Cause                     | Solution                        |
| ----------------------- | ------------------------------ | ------------------------------- |
| `ImagePullBackOff`      | Tag does not exist in registry | Use a valid, existing image tag |
| `ErrImagePull` persists | Tag deprecated or deleted      | Update to a supported version   |
