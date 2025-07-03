# 🛠️ Kubernetes PV, PVC, SC & StatefulSet Troubleshooting Guide

This guide provides a concise overview of five common issues when working with Kubernetes **Persistent Volumes (PVs)**, **Persistent Volume Claims (PVCs)**, **StorageClasses (SCs)**, and **StatefulSets (STSs)** — along with quick commands and fixes.

---

## ❌ 1. PVC Stuck in `Pending`

**🔍 Root Cause:**
- The `StorageClass` does not exist or has an incorrect `provisioner`
- Dynamic provisioning is not working

**✅ Quick Fix:**
```bash
kubectl get sc                    # Check available StorageClasses
kubectl describe pvc <pvc-name>   # Review events and SC used
```
Ensure a valid `StorageClass` exists (e.g., `ebs.csi.aws.com` for AWS).

---

## ❌ 2. PV and PVC Not Bound

**🔍 Root Cause:**
- Mismatched specs between PV and PVC:
  - StorageClassName
  - AccessModes
  - Requested capacity

**✅ Quick Fix:**
```bash
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>
```
Ensure exact match between:
- `storageClassName`
- `accessModes`
- `resources.requests.storage`

---

## ❌ 3. StatefulSet Pod Stuck in `ContainerCreating`

**🔍 Root Cause:**
- PVC is stuck in `Pending`
- Invalid or missing StorageClass

**✅ Quick Fix:**
```bash
kubectl get pvc | grep <sts-name>
kubectl describe sts <sts-name>
```
Check if the associated PVC is bound and StorageClass is correct.
Fix or recreate SC → redeploy the StatefulSet.

---

## ❌ 4. Pod Fails to Mount Volume

**🔍 Root Cause:**
- PVC name typo in YAML
- PV is in `Released` state and cannot bind
- CSI driver not running

**✅ Quick Fix:**
```bash
kubectl describe pod <pod-name>
kubectl get pods -n kube-system | grep csi
```
- Fix YAML PVC name
- Restart failed CSI pods (e.g., `ebs-csi-*`)
- Consider deleting PV finalizers if stuck

---

## ❌ 5. PV in `Released` State Not Reusable

**🔍 Root Cause:**
- Previous PVC was deleted
- PV is `Retain` but has no claim bound

**✅ Quick Fix:**
```bash
kubectl patch pv <pv-name> -p '{"metadata":{"finalizers":null}}'
```
- Remove finalizer
- Create a new PVC matching its definition

---

## ✅ Tips for StatefulSet + EBS + PVC Success

- Always set `volumeBindingMode: WaitForFirstConsumer` in SC
- Use `Retain` policy for StatefulSets needing volume recovery
- Track PVC names (`<claim-name>-<sts-name>-<ordinal>`) carefully

---

## 📎 Reference Commands

```bash
kubectl get pvc,pv
kubectl get sc
kubectl describe pvc <name>
kubectl describe pv <name>
kubectl get sts,pods
kubectl logs <pod>                 # Check mount/volume errors
kubectl get pods -n kube-system    # Check CSI status
```
# 🧰 EKS StatefulSet PVC Recovery Lab (AWS EBS CSI)

## ✅ Step 0: Create EKS Cluster

Using `eksctl`:

```bash
eksctl create cluster \
  --name ebs-demo-cluster \
  --region us-east-1 \
  --zones us-east-1a,us-east-1b \
  --version 1.27 \
  --nodegroup-name linux-nodes \
  --node-type t2.medium \
  --nodes 2 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key yrr-demo-key
```

⏳ Wait for ~15 minutes

Verify cluster:
```bash
kubectl get nodes
```

---

## 🔌 Step 1: Verify EBS CSI Driver

Check if it's running:
```bash
kubectl get pods -n kube-system | grep ebs
```

If not, install manually:
```bash
eksctl create addon --name aws-ebs-csi-driver --cluster ebs-demo-cluster --region us-east-1 --force
```

---

## ⚠️ Step 2: Simulate Broken PVC

### 2.1 Create Invalid StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: broken-sc
provisioner: fake.aws.ebs
reclaimPolicy: Retain
```
```bash
kubectl apply -f fake-sc.yaml
kubectl get storageclass
```

### 2.2 Deploy Broken StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-broken
spec:
  serviceName: "web"
  replicas: 1
  selector:
    matchLabels:
      app: web-broken
  template:
    metadata:
      labels:
        app: web-broken
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: web-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: broken-sc
      resources:
        requests:
          storage: 1Gi
```
```bash
kubectl apply -f broken-sts.yaml
kubectl get sts
kubectl describe sts web-broken
kubectl get pvc
kubectl describe pvc web-data-web-broken-0
```

PVC will be stuck in `Pending`.

---

## 🧹 Step 3: Fix Configuration

### 3.1 Delete Broken Resources
```bash
kubectl delete sts web-broken
kubectl delete pvc web-data-web-broken-0
kubectl delete sc broken-sc
```

### 3.2 Create Valid StorageClass (EBS CSI)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```
```bash
kubectl apply -f ebs-sc.yaml
kubectl get sc
```

### 3.3 Deploy Fixed StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: web-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ebs-sc
      resources:
        requests:
          storage: 1Gi
```
```bash
kubectl apply -f sts.yaml
kubectl get sts
kubectl get pv
```

Confirm volume is created in AWS Console.
```bash
kubectl exec web-0 -- /bin/sh -c "echo 'Hello from EKS EBS' > /usr/share/nginx/html/index.html"
```

---

## 💾 Step 4: Snapshot and Restore EBS Volume

### 4.1 Get Volume ID
```bash
kubectl get pv
kubectl describe pv <PV-NAME>
```
Look for `volumeHandle`, e.g., `vol-0abc123xyz`.

### 4.2 Create Snapshot
```bash
aws ec2 create-snapshot --volume-id vol-XXXXXXXXXXXXXXX --description "Snapshot for StatefulSet restore"
aws ec2 describe-snapshots --owner-ids self --region us-east-1
```
Note the Snapshot ID.

### 4.3 Create New Volume from Snapshot
```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --snapshot-id snap-XXXXXXXXXXXXXXX \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=web-restored}]'
```
Note new Volume ID (e.g. `vol-0restored123456`).

---

## 🧱 Step 5: Restore StatefulSet from Snapshot

### 5.1 Create PersistentVolume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: web-restored-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ebs-sc
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-041a0b94759049706
    fsType: ext4
```
```bash
kubectl apply -f pv.yaml
kubectl get pv
```

### 5.2 Create Matching PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data-web-restore-0
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 1Gi
```
```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

### 5.3 Deploy Restored StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-restore
spec:
  serviceName: "web"
  replicas: 1
  selector:
    matchLabels:
      app: web-restore
  template:
    metadata:
      labels:
        app: web-restore
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: web-data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: web-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ebs-sc
        resources:
          requests:
            storage: 1Gi
```
```bash
kubectl apply -f restore-sts.yaml
kubectl get sts
kubectl get pods
kubectl exec web-restore-0 -- cat /usr/share/nginx/html/index.html
```
Expected output:
```
Hello from EKS EBS
```

---

## 🧹 Optional Cleanup
```bash
eksctl delete cluster --name ebs-demo-cluster --region us-east-1
```

---

## ✅ Final Summary
| Step | Description |
|------|-------------|
| 0 | Created EKS cluster with eksctl |
| 1 | Installed/verified EBS CSI driver |
| 2 | Simulated PVC unbound with invalid SC |
| 3 | Deployed working StatefulSet with EBS SC |
| 4 | Snapshotted and restored EBS volume |
| 5 | Recovered StatefulSet using restored volume |
