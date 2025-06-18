1️⃣ Scenario: Wrong Image Name or Tag

A common typo like nginx:latets (instead of nginx:latest) can prevent Kubernetes from pulling the container image.

🔧 Simulate the Issue

Step 1: Create a broken Pod manifest

# wrong-image.yaml
apiVersion: v1
kind: Pod
metadata:
  name: wrong-image
spec:
  containers:
  - name: app
    image: nginx:latets

Step 2: Apply the manifest

kubectl apply -f wrong-image.yaml

Step 3: Check Pod status

kubectl get pods

Expected output:

NAME          READY   STATUS         RESTARTS   AGE
wrong-image   0/1     ErrImagePull   0          20s

🔍 Analyze the Problem

Use kubectl describe to view the detailed error:

kubectl describe pod wrong-image

Sample Events Output:

Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  100s                default-scheduler  Successfully assigned default/wrong-image to node01
  Normal   Pulling    51s (x3 over 100s)  kubelet            Pulling image "nginx:latets"
  Warning  Failed     44s (x3 over 91s)   kubelet            Failed to pull image "nginx:latets": ... not found
  Warning  Failed     44s (x3 over 91s)   kubelet            Error: ErrImagePull
  Normal   BackOff    5s (x5 over 91s)    kubelet            Back-off pulling image "nginx:latets"
  Warning  Failed     5s (x5 over 91s)    kubelet            Error: ImagePullBackOff

🧐 What Happened?

Kubernetes initially failed to pull the image → ErrImagePull

After multiple failed attempts → ImagePullBackOff (starts retrying with delay)

🛠️ Fix the Problem

Step 1: Edit the Pod

kubectl edit pod wrong-image

Step 2: Correct the image line

Replace:

image: nginx:latets

With:

image: nginx:latest

Step 3: Verify the fix

kubectl get pods

Expected output:

NAME          READY   STATUS    RESTARTS   AGE
wrong-image   1/1     Running   0          Xm

✅ Summary

Problem

Solution

Typo in image tag (latets)

Correct it to latest

Pod stuck in ImagePullBackOff

Edit and fix the image name
