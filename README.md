# C-DAC Kubernetes Demo: Static Website on Minikube

A minimal demo of deploying, scaling, updating and rolling back a static website on Kubernetes.

| Version | What it shows |
|---------|---------------|
| v1 | "Welcome to C-DAC" in **blue** |
| v2 | Text in **red**, plus a second line "This is Kubernetes" |

```
kubernetes-static/
├── v1/              Dockerfile + index.html (blue)
├── v2/              Dockerfile + index.html (red, extra line)
├── k8s/
│   └── deployment.yaml   Deployment (3 replicas) + NodePort Service (port 30080)
└── README.md
```

---

## 0. Prerequisites (Ubuntu)

- Docker installed, and your user can run it without `sudo`:
  ```bash
  sudo usermod -aG docker $USER && newgrp docker
  ```
- `minikube` and `kubectl` installed

Check:
```bash
docker version
minikube version
kubectl version --client
```

Start the cluster:
```bash
minikube start --driver=docker
kubectl get nodes          # should show one node in Ready state
```

---

## 1. Build the v1 image inside minikube

Minikube runs its own Docker daemon. If you build images on your host Docker, the cluster
**cannot see them** and pods fail with `ErrImagePull`. So point your shell at minikube's Docker first:

```bash
eval $(minikube docker-env)
```

> This applies only to the current terminal. Run it again in any new terminal.

Build v1:
```bash
docker build -t cdac-web:v1 ./v1
docker images | grep cdac-web
```

---

## 2. Deploy v1

```bash
kubectl apply -f k8s/deployment.yaml
```

Check what got created:
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

## 3. Explore

```bash
kubectl describe deployment cdac-web
kubectl describe pod <pod-name>
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

## 4. Rolling update to v2

Build the v2 image (make sure `eval $(minikube docker-env)` is active in this terminal):
```bash
docker build -t cdac-web:v2 ./v2
```

In a second terminal, watch the pods:
```bash
kubectl get pods -w
```

Update the image:
```bash
kubectl set image deployment/cdac-web web=cdac-web:v2
kubectl rollout status deployment/cdac-web
```

Kubernetes starts new pods and removes old ones **one at a time**, so the site stays up
the whole time (zero downtime). Refresh the browser: the text is now **red**, with the
extra line **"This is Kubernetes"**.

> You may need a hard refresh (Ctrl+Shift+R) because of browser caching.

### Alternative: the declarative way (recommended in real projects)
Edit `k8s/deployment.yaml` and change `image: cdac-web:v1` to `image: cdac-web:v2`, then:
```bash
kubectl apply -f k8s/deployment.yaml
```
This does the same rolling update, but the YAML file remains the single source of truth.

---

## 5. Rollout history and rollback

```bash
kubectl rollout history deployment/cdac-web
kubectl rollout undo deployment/cdac-web
kubectl rollout status deployment/cdac-web
```

Refresh the browser: it's back to **blue** v1.

---

## 6. Clean up

```bash
kubectl delete -f k8s/deployment.yaml
eval $(minikube docker-env -u)   # point the shell back to host Docker
minikube stop
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Pod status `ErrImagePull` / `ImagePullBackOff` | Image was built on host Docker. Run `eval $(minikube docker-env)` and rebuild. |
| Browser shows old page after update | Hard refresh (Ctrl+Shift+R) or open in a private window. |
| `minikube ip:30080` not reachable | Use `minikube service cdac-web --url` instead. |
| `permission denied` on docker | Add user to docker group (see Prerequisites). |
