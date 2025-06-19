# 🐳 CrashLoopBackOff in Kubernetes

## What is CrashLoopBackOff?

`CrashLoopBackOff` is a **Pod status** in Kubernetes that means the container is **repeatedly crashing after starting**. Kubernetes tries to restart it, but it keeps failing — entering a loop of:

crash → restart → crash
---

## Why Does It Happen?

Here are the most common causes:

| Cause                      | Description                                                     |
|---------------------------|-----------------------------------------------------------------|
| **Application bug**       | App exits with a non-zero exit code                            |
| **Misconfiguration**      | Missing environment variables, wrong command/args              |
| **Secret/ConfigMap issue**| Referenced file or key doesn't exist                           |
| **Health probe failure**  | Liveness/Readiness probe misfires, triggering restarts        |
| **Permission error**      | File or volume access denied                                   |
| **OOMKilled**             | Container exceeds memory limits and gets killed               |
| **Expected exit**         | App exits normally but was deployed as a Deployment (e.g., job)|

---

## 🛠️ How to Fix It

### 🔹 1. Fix App Logic / Code
Ensure the application doesn’t crash on startup. Handle exceptions and errors properly.

### 🔹 2. Correct Environment Variables or Args
Check and define all required:
- `env` variables
- container `command` or `args`

### 🔹 3. Check ConfigMap/Secret Mounts
Make sure your configuration is correct:
```yaml
envFrom:
  - configMapRef:
      name: my-config
```
or
```
volumeMounts:
  - name: my-secret
    mountPath: /etc/secret
```
✅ Ensure referenced resources exist and are mounted correctly.
### 🔹 4. Fix Health Probes
Example of correct liveness probe:
```
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```
💡 You can disable probes temporarily for debugging.
### 🔹 5. Adjust Memory Limits
Check for OOMKilled status and adjust limits:
```
resources:
  limits:
    memory: "256Mi"
```
### 🔹 6. Increase Startup Delay
If your app takes time to boot, increase:
```
livenessProbe.initialDelaySeconds
```
### 🔹 7. Use Init Containers
Use initContainers to wait for external dependencies (e.g., DB, API) before starting the main app.
## ✅ Best Practice Tip
💡 Always test your image before deploying:
```
docker run your-image
```
