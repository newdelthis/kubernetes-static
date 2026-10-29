# C-DAC Kubernetes Demo: Static Website on Minikube

A minimal demo of deploying, scaling, updating and rolling back a static website on Kubernetes.

The images are **already on Docker Hub**, so the cluster just pulls them. No image building is needed during the demo.

| Version | Image | What it shows |
|---------|-------|---------------|
| v1 | `newdelthis/cdac-web:v1` | "Welcome to C-DAC" in **blue** |
| v2 | `newdelthis/cdac-web:v2` | Text in **red**, plus a second line "This is Kubernetes" |

```
kubernetes-static/
├── v1/              Dockerfile + index.html (blue)        - source of the v1 image
├── v2/              Dockerfile + index.html (red, extra line) - source of the v2 image
├── k8s/
│   └── deployment.yaml   Deployment (3 replicas) + NodePort Service (port 30080)
└── README.md
```

---

## 0. Prerequisites (Ubuntu)

- `minikube` and `kubectl` installed
- Docker installed (minikube uses it as its driver), and your user can run it without `sudo`:
  ```bash
  sudo usermod -aG docker $USER && newgrp docker
  ```
- Internet access (to pull the images from Docker Hub)

Start the cluster:
```bash
minikube start --driver=docker
kubectl get nodes          # should show one node in Ready state
```

---

## 1. Deploy v1

```bash
kubectl apply -f k8s/deployment.yaml
```

Kubernetes pulls `newdelthis/cdac-web:v1` from Docker Hub and starts 3 pods. Check:
```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get svc cdac-web
```

Open in browser:
```bash
minikube service cdac-web --url
# or
echo "http://$(minikube ip):30080"
```

You should see **"Welcome to C-DAC"** in blue.

---

## 2. Explore

```bash
kubectl describe deployment cdac-web
kubectl describe pod <pod-name>     # see the "Pulling image" / "Pulled" events
kubectl logs <pod-name>
```

**Self-healing**: delete a pod and watch Kubernetes replace it.
```bash
kubectl get pods -w        # in terminal 2
kubectl delete pod <pod-name>
```

**Scaling**:
```bash
kubectl scale deployment cdac-web --replicas=5
kubectl get pods
kubectl scale deployment cdac-web --replicas=3
```

---

## 3. Rolling update to v2

In a second terminal, watch the pods:
```bash
kubectl get pods -w
```

Update the image:
```bash
kubectl set image deployment/cdac-web web=newdelthis/cdac-web:v2
kubectl rollout status deployment/cdac-web
```

Kubernetes starts new pods and removes old ones **one at a time**, so the site stays up
the whole time (zero downtime). Refresh the browser: the text is now **red**, with the
extra line **"This is Kubernetes"**.

> You may need a hard refresh (Ctrl+Shift+R) because of browser caching.

### Alternative: the declarative way (recommended in real projects)
Edit `k8s/deployment.yaml` and change `newdelthis/cdac-web:v1` to `newdelthis/cdac-web:v2`, then:
```bash
kubectl apply -f k8s/deployment.yaml
```
This does the same rolling update, but the YAML file remains the single source of truth.

---

## 4. Rollout history and rollback

```bash
kubectl rollout history deployment/cdac-web
kubectl rollout undo deployment/cdac-web
kubectl rollout status deployment/cdac-web
```

Refresh the browser: it's back to **blue** v1.

---

## 5. Clean up

```bash
kubectl delete -f k8s/deployment.yaml
minikube stop
```

---

## Appendix: how the images were built (one-time, not part of the demo)

Done once on any machine with Docker, logged in to Docker Hub (`docker login`):
```bash
docker build -t newdelthis/cdac-web:v1 ./v1
docker build -t newdelthis/cdac-web:v2 ./v2
docker push newdelthis/cdac-web:v1
docker push newdelthis/cdac-web:v2
```
To use your own Docker Hub account instead, replace `newdelthis` with your username here
and in `k8s/deployment.yaml`.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Pod status `ErrImagePull` / `ImagePullBackOff` | Check internet access from minikube (`minikube ssh -- curl -sI https://registry-1.docker.io`) and the image name/tag spelling. Run `kubectl describe pod <pod-name>` for the exact error. |
| `toomanyrequests` in pod events | Docker Hub anonymous pull limit hit. Wait, or run `docker login` and recreate the cluster. |
| Browser shows old page after update | Hard refresh (Ctrl+Shift+R) or open in a private window. |
| `minikube ip:30080` not reachable | Use `minikube service cdac-web --url` instead. |
| `permission denied` on docker | Add user to docker group (see Prerequisites). |
