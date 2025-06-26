# argocd

This repository contains an end-to-end setup to run ArgoCD in a local Minikube cluster with:

- ✅ GitOps via ArgoCD App of Apps pattern
- 🎯 GitHub Action triggered on PostSync (hello world)
- 📊 Prometheus + Grafana monitoring with custom dashboards
- 🔐 Vault integration for secrets

---

## 📦 Folder Structure

```bash
.
├── apps-of-apps.yaml           # Main ArgoCD App of Apps definition
├── argoapp.yaml                # Sample ArgoCD application
├── deploy-files/testing-applications/
├── gitea.yaml                  # Optional Git repo hosting
├── hashicorp-vault/            # Vault setup with integration
├── secret.yaml                 # Generic secret manifest
├── test-prometheus/            # Prometheus service monitor configs
├── vault-secret.yaml
├── vault.yaml
└── README.md
```

---

## 🧪 Requirements

- Minikube (v1.30+)
- kubectl
- Helm
- GitHub PAT (for triggering GitHub Actions from ArgoCD hooks)

---

## 🧰 Setup Instructions

### 🟢 1. Start Minikube

```bash
minikube start --cpus=4 --memory=6g
```

---

### 🟦 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to be ready:

```bash
kubectl get pods -n argocd
```

---

### 🟨 3. Expose ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login:

```bash
argocd login localhost:8080
# Default username: admin
# Password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

---

### 🧩 4. Apply App-of-Apps

```bash
kubectl apply -f apps-of-apps.yaml
```

This sets up all the sub-applications like Vault, Prometheus stack, and your test workloads.

---

## 📈 Prometheus + Grafana Monitoring

### 🛠 What is Prometheus?

Prometheus is an open-source monitoring system that collects metrics from configured targets at given intervals, evaluates rule expressions, displays the results, and can trigger alerts if conditions are met.

### 🛠 What is Grafana?

Grafana is an open-source visualization and analytics software. It allows you to query, visualize, and understand your metrics from Prometheus.

### 🛠 What is a ServiceMonitor?

A `ServiceMonitor` is a custom resource used by the Prometheus Operator to define how a service should be monitored. You specify the namespace, selector labels, and endpoints to scrape metrics from.

### ⚙️ Prometheus Setup

Install Prometheus Operator:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

Apply your ServiceMonitor configurations:

```bash
kubectl apply -f test-prometheus/
```

Ensure your target services have metrics endpoints (e.g., `/metrics`).

### 📊 Access Grafana UI

```bash
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
```

Access: [http://localhost:3000](http://localhost:3000)

- Default user: `admin`
- Password:

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

You can import dashboards from JSON or define them as ConfigMaps inside `Grafana-Dashboards/`.

---

## 🟩 GitHub Action Triggered via ArgoCD PostSync Hook

### 1. GitHub Workflow Setup

Create `.github/workflows/hello-world.yml` in your GitHub repo:

```yaml
name: Hello World Workflow

on:
  workflow_dispatch:

jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
      - name: Say Hello
        run: echo "Hello, World!"
```

### 2. Secret Setup

Store your GitHub PAT (with `workflow` scope):

```bash
kubectl create secret generic github-token \
  --from-literal=token=<your_pat_here> \
  -n default
```

### 3. Add PostSync Hook

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: trigger-github-action
  annotations:
    "argocd.argoproj.io/hook": PostSync
    "argocd.argoproj.io/hook-delete-policy": HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: trigger
          image: curlimages/curl
          command: ["curl"]
          args:
            - -X
            - POST
            - -H
            - "Accept: application/vnd.github+json"
            - -H
            - "Authorization: Bearer $GITHUB_TOKEN"
            - https://api.github.com/repos/<your-username>/argocd/actions/workflows/hello-world.yml/dispatches
            - -d
            - '{"ref":"main"}'
          env:
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: github-token
                  key: token
      restartPolicy: Never
```

---

---

## 🧼 Cleanup

```bash
minikube delete
```

---

## 🤝 Contributing

Feel free to fork and suggest improvements. Star this repo if it helped you.

---

## 📜 License

MIT

