## 5️⃣ Scenario: Incorrect imagePullSecrets

If the `imagePullSecrets` field in a Pod spec is incorrect (wrong name, missing, or misconfigured), Kubernetes cannot authenticate to the private registry — resulting in an `ImagePullBackOff`.

### 🔍 Why It Happens
- The referenced secret name does **not exist** in the namespace.
- The secret is not of type `kubernetes.io/dockerconfigjson`.
- The credentials in the secret are **invalid or expired**.

### 🛠️ Fix
- Ensure the secret exists:
  ```bash
  kubectl get secrets
  ```
- If not, create it:
  ``` bash 
  kubectl create secret docker-registry regcred \
  --docker-username=<your-docker-id> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
  ```
- Make sure it’s correctly referenced in the Pod:
```
imagePullSecrets:
- name: regcred
```
## ✅ Result
Once the correct secret is applied, the Pod should transition to Running after pulling the private image successfully.
