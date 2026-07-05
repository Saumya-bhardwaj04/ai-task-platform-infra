# AI Task Platform - Infrastructure Repository

This repo holds every Kubernetes manifest for the AI Task Processing Platform. Argo CD watches this repo and applies whatever is in the `k8s` folder to the cluster automatically (GitOps).

## Folder structure

```
k8s/            all Kubernetes manifests
argocd/         the Argo CD Application definition
```

## Prerequisites on your cluster

- A running k3s (or any Kubernetes) cluster
- `kubectl` configured to point at it
- An ingress controller installed (ingress-nginx)
- metrics-server installed (required for the worker's HPA to read CPU usage)
- Argo CD installed in the cluster

## One-time manual setup (before Argo CD takes over)

1. Create the real secret (never commit this file — it's git-ignored on purpose):
```bash
cp k8s/secret.example.yaml k8s/secret.yaml
# edit k8s/secret.yaml and put your real MONGO_URI and JWT_SECRET in it
kubectl apply -f k8s/secret.yaml
```

2. Apply everything else once manually, just to confirm it all works before wiring up Argo CD:
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/redis-service.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/backend-service.yaml
kubectl apply -f k8s/worker-deployment.yaml
kubectl apply -f k8s/worker-hpa.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/frontend-service.yaml
kubectl apply -f k8s/ingress.yaml
```

3. Check everything is running:
```bash
kubectl get pods -n ai-task-platform
kubectl get ingress -n ai-task-platform
```

## Setting up Argo CD (GitOps)

1. Install Argo CD (skip if already installed on the cluster):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Get the initial admin password and log in:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080, user: admin, password: from the command above
```

3. Point Argo CD at this repo by applying the Application definition:
```bash
kubectl apply -f argocd/application.yaml
```

That's it. From now on, whenever the `k8s` folder in this repo changes (for example, the CI/CD pipeline in the app repo updates an image tag after a new build), Argo CD notices the diff and automatically applies it to the cluster because `syncPolicy.automated` is turned on. You can watch it happen live in the Argo CD dashboard.

## Worker scaling

The worker has a HorizontalPodAutoscaler (`worker-hpa.yaml`) that scales it between 2 and 10 pods based on CPU usage. Since each worker pod pulls independently from the same Redis queue, adding more pods directly increases how many tasks are processed in parallel — no code changes needed.
