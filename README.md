# AI-Chat GitOps

This repository contains the Kubernetes manifests for the AI-Chat project, automatically deployed through Argo CD using the GitOps pattern. Any change pushed to `main` is detected and applied by Argo CD to the K3s cluster.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Deployment](#deployment)
- [Manifests](#manifests)
- [AI Models](#ai-models)
- [Updating the Image](#updating-the-image)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This GitOps repository orchestrates the deployment of two main components on Kubernetes:

- **Ollama** — AI inference server with 3 models (llama3:8b, llava:7b, gemma4:12b)
- **Next.js frontend** — Chat interface exposed through a Traefik Ingress

The CI/CD pipeline in the [AI-Chat-app](https://github.com/CL-K/AI-Chat-app) repository automatically updates the image tag in `frontend.yaml` on every push to `main`, triggering an Argo CD deployment.

## 🏗️ Architecture

```
                        Internet
                            │
                            ▼
                   Traefik Ingress
                   (ai-chat.com:80)
                            │
                            ▼
                ┌───────────────────────┐
                │  Namespace: ai-chat   │
                │                       │
                │  ┌─────────────────┐  │
                │  │ Frontend        │  │
                │  │ Next.js         │  │
                │  │ port 3000→80    │  │
                │  └────────┬────────┘  │
                │           │           │
                │           ▼           │
                │  ┌─────────────────┐  │
                │  │ Ollama          │  │
                │  │ port 11434      │  │
                │  │                 │  │
                │  │ llama3:8b       │  │
                │  │ llava:7b        │  │
                │  │ gemma4:12b      │  │
                │  └────────┬────────┘  │
                │           │           │
                │  ┌────────▼────────┐  │
                │  │ PVC 30Gi        │  │
                │  │ (AI models)     │  │
                │  └─────────────────┘  │
                └───────────────────────┘
```

## 📦 Prerequisites

- Operational K3s cluster (at least 3 nodes)
- Argo CD installed on the cluster
- Traefik installed (included by default with K3s)
- `ai-chat.com` DNS pointing to the master's public IP address
- Node with at least **32GB RAM** for Ollama

## 📁 Repository Structure

```
AI-Chat-gitops/
├── namespace.yaml    # ai-chat namespace
├── ollama.yaml       # Ollama Deployment + PVC + Service
├── frontend.yaml     # Next.js Deployment + Service
├── ingress.yaml      # Traefik Ingress (ai-chat.com)
└── README.md
```

## 🚀 Deployment

### Manual Deployment (First Time)

```bash
# 1. Create the namespace
kubectl apply -f namespace.yaml

# 2. Deploy Ollama (model downloads take 20–30 minutes)
kubectl apply -f ollama.yaml

# 3. Deploy the frontend
kubectl apply -f frontend.yaml

# 4. Configure the Ingress
kubectl apply -f ingress.yaml
```

### Automatic Deployment Through Argo CD

After the initial setup, Argo CD monitors this repository and automatically applies any changes pushed to `main`.

**Configure Argo CD:**

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ai-chat
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/CL-K/AI-Chat-gitops
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: ai-chat
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

## 📄 Manifests

### namespace.yaml

Creates the `ai-chat` namespace with `project` and `environment` labels to make it easier to filter resources.

### ollama.yaml

Contains three resources:

**Ollama Deployment** with two containers:
- `initContainer (pull-models)` — downloads the 3 models on first startup before starting the server. If a pull fails, the pod remains in `Init:Error`, making the failure explicit and detectable.
- `Container (ollama)` — runs `ollama serve` in the foreground. If the process crashes, Kubernetes automatically restarts the pod.

**PersistentVolumeClaim** — 30Gi to store the models (~16.6GB total, plus extra capacity).

**Service** — exposes Ollama on port `11434` within the cluster.

### frontend.yaml

**Next.js Deployment** with:
- Image automatically updated by the CI pipeline on every push
- `OLLAMA_API_URL=http://ollama:11434` environment variable
- `readinessProbe` — Kubernetes waits for Next.js to be ready before sending traffic
- `livenessProbe` — Kubernetes restarts the pod if Next.js stops responding
- Resources limited to 512Mi RAM / 500m CPU

**Service** — maps port `80` to the container's port `3000`.

### ingress.yaml

Exposes the frontend at `ai-chat.com` through Traefik (`ingressClassName: traefik`). All incoming traffic on `/` is routed to the `ai-chat-frontend` Service on port `80`.

## 🤖 AI Models

| Model | Size | Use |
|---|---|---|
| `llama3:8b` | ~5GB | Text generation (route `/api/llama`) |
| `llava:7b` | ~4GB | Text and image analysis (route `/api/llava`) |
| `gemma4:12b` | ~7.6GB | Multimodal text/image/audio/video (route `/api/gemma`) |

> **Note:** The first Ollama startup takes **20–30 minutes** to download the 3 models (~16.6GB). Subsequent startups are immediate because the models are persisted in the PVC.

## 🔄 Updating the Image

The frontend image tag is automatically updated by the CI pipeline in the [AI-Chat-app](https://github.com/CL-K/AI-Chat-app) repository using this pattern:

```bash
# The CI pipeline runs this command after a successful build
sed -i "s|ghcr.io/cl-k/ai-chat-app:.*|ghcr.io/cl-k/ai-chat-app:<SHA>|g" frontend.yaml
git commit -m "chore: update ai-chat-app image to <SHA>"
git push origin main
```

Argo CD detects the new commit and automatically deploys the new image.

**Complete workflow:**
```
Push to AI-Chat-app
  → GitHub Actions (lint → tests → build → Trivy scan)
  → Push image to GHCR
  → Update frontend.yaml in this repository
  → Argo CD detects the change
  → Deploy to K3s
```

## 🔧 Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n ai-chat
kubectl get pods -n ai-chat -w  # watch in real time
```

### Ollama Stuck in Init:Error

```bash
# View the init container logs
kubectl logs -n ai-chat -l app=ollama -c pull-models

# Possible causes:
# - Network unavailable (pull failed)
# - Disk full (check the PVC)
# - Model not found in the Ollama registry
```

### Ollama Stuck in Pending

```bash
kubectl describe pod -n ai-chat -l app=ollama

# Possible causes:
# - Not enough RAM available on the node (16Gi required)
# - PVC cannot be mounted (different node)
```

### Frontend Returns 500 Errors

```bash
# Check that Ollama is Running
kubectl get pods -n ai-chat

# Check the frontend logs
kubectl logs -n ai-chat -l app=ai-chat-frontend

# Test the Ollama API directly from the cluster
kubectl exec -n ai-chat deploy/ollama -- curl -s http://localhost:11434/api/tags
```

### Ingress Not Responding

```bash
# Check that Traefik is running
kubectl get pods -n kube-system | grep traefik

# Check the Ingress
kubectl describe ingress -n ai-chat ai-chat-ingress

# Check that DNS points to the correct node
nslookup ai-chat.com
```

### List Installed Ollama Models

```bash
kubectl exec -n ai-chat deploy/ollama -- ollama list
```

### Force an Argo CD Re-sync

```bash
argocd app sync ai-chat
# or click "Sync" in the Argo CD interface
```

## 📚 Related Repositories

- [AI-Chat-app](https://github.com/CL-K/AI-Chat-app) — Next.js source code + CI/CD pipeline
- [AI-Chat-infra](https://github.com/CL-K/AI-Chat-infra) — AWS infrastructure (K3s cluster)

## 📝 License

This project is licensed under the MIT License.
See the [LICENSE](LICENSE) file for more details.

## 👤 Author

Created by Cheick — June 2026
