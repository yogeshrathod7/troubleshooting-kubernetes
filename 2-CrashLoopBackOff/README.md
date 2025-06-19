# 🐳 CrashLoopBackOff in Kubernetes

## ❓ What is CrashLoopBackOff?

`CrashLoopBackOff` is a **Pod status** in Kubernetes that means the container is **repeatedly crashing after starting**. Kubernetes tries to restart it, but it keeps failing — entering a loop of:

crash → restart → crash
