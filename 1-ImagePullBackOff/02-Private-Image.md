## 2️⃣ Scenario: Private Image on Docker Hub

We will simulate an image pull failure using a private image hosted on Docker Hub and fix it by using a Kubernetes `imagePullSecret`.

---
### 🧪 Setup: Push a Private Image to Docker Hub

Make sure you're logged in to Docker:

```bash
docker pull yogeshrathod1137/nginx-custom:01
docker tag yogeshrathod1137/nginx-custom:01 yogeshrathod1137/nginx-private
docker push yogeshrathod1137/nginx-private
```

🔐 Then, go to Docker Hub and make the yogeshrathod1137/nginx-private repository private.
### 📄 Step 1: Create a Pod Without Credentials
Create private-registry.yaml:
```
apiVersion: v1
kind: Pod
metadata:
  name: private-image
spec:
  containers:
  - name: app
    image: yogeshrathod1137/nginx-private
```
Apply it:
```
kubectl apply -f private-registry.yaml
```
Check Pod status:
```
kubectl get pods
```
Expected Output:
```
NAME            READY   STATUS             RESTARTS   AGE
private-image   0/1     ImagePullBackOff   0          2m45s
```
## 🔍 Analyze the Error
```
kubectl describe pod private-image
```
Sample Events Output:
```
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  ...                 default-scheduler  Successfully assigned
  Normal   Pulling    ...                 kubelet            Pulling image "yogeshrathod1137/nginx-private"
  Warning  Failed     ...                 kubelet            Failed to pull image: pull access denied
  Warning  Failed     ...                 kubelet            Error: ErrImagePull
  Normal   BackOff    ...                 kubelet            Back-off pulling image
  Warning  Failed     ...                 kubelet            Error: ImagePullBackOff
```
## 🛠️ Fix: Use Docker Registry Credentials
### Step 1: Create a Secret for Registry Access
```
kubectl create secret docker-registry regcred \
  --docker-username=yourdockerid \
  --docker-password=yourpassword \
  --docker-email=youremail@example.com
```
🔐 Replace values with your actual Docker Hub credentials.
### Step 2: Update the Pod to Use the Secret
Edit the pod definition and add imagePullSecrets:
```
apiVersion: v1
kind: Pod
metadata:
  name: private-image
spec:
  containers:
  - name: app
    image: yogeshrathod1137/nginx-private
  imagePullSecrets:
  - name: regcred
```
Save as private-registry.yaml.
### Step 3: Recreate the Pod
```
kubectl delete pod private-image
kubectl apply -f private-registry.yaml
```
### Step 4: Verify
```
kubectl get pods
```
Expected Output:
```
NAME            READY   STATUS    RESTARTS   AGE
private-image   1/1     Running   0          89s
```
## ✅ Summary
```
| Problem                        | Root Cause                         | Solution                           |
| ------------------------------ | ---------------------------------- | ---------------------------------- |
| `ImagePullBackOff`             | Private image pull failed (401)    | Use `imagePullSecrets` with Docker |
| `ErrImagePull` + access denied | Missing or incorrect registry auth | Create and reference secret        |
```
