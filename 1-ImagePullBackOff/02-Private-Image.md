## 2️⃣ Scenario: Private Image on Docker Hub

We will simulate an image pull failure using a private image hosted on Docker Hub and fix it by using a Kubernetes `imagePullSecret`.

---
## 🧪 Setup: Push a Private Image to Docker Hub

Make sure you're logged in to Docker:

```bash
docker pull yogeshrathod1137/nginx-custom:01
docker tag yogeshrathod1137/nginx-custom:01 yogeshrathod1137/nginx-private
docker push yogeshrathod1137/nginx-private

🔐 Then, go to Docker Hub and make the yogeshrathod1137/nginx-private repository private.
