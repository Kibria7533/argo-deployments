# Install helm
```sh
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
# Install Argocd
# 🚀 GitHub Actions: Access EKS and Install ArgoCD

This document describes how to use a GitHub Actions workflow to:

1. Access an **Amazon EKS** cluster
2. Install **ArgoCD** if it’s not already installed

---

## 📦 Prerequisites

### ✅ GitHub Secrets
You must configure the following secrets in your GitHub repository:

| Secret Name            | Description                          |
|------------------------|--------------------------------------|
| `AWS_ACCESS_KEY_ID`    | Your AWS access key ID               |
| `AWS_SECRET_ACCESS_KEY`| Your AWS secret access key           |
| `AWS_REGION` *(optional)*       | AWS region (e.g., `us-east-1`)     |

---

## 🛠️ Workflow File

Save the following YAML content as `.github/workflows/eks-argocd-setup.yml`:

```yaml
name: Access EKS and Install ArgoCD

on:
  workflow_dispatch: # Manually triggered

jobs:
  install-argocd:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Install kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Update kubeconfig for EKS
        run: |
          aws eks update-kubeconfig --name <your-cluster-name> --region us-east-1

      - name: Check if ArgoCD is already installed
        id: check-argocd
        run: |
          if kubectl get namespace argocd > /dev/null 2>&1; then
            echo "ArgoCD is already installed"
            echo "exists=true" >> $GITHUB_OUTPUT
          else
            echo "ArgoCD not found"
            echo "exists=false" >> $GITHUB_OUTPUT
          fi

      - name: Install ArgoCD
        if: steps.check-argocd.outputs.exists == 'false'
        run: |
          kubectl create namespace argocd
          kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


# ArgoCD App of Apps Setup for nginx-ingress

This repository uses the **App of Apps** pattern in ArgoCD to manage Helm chart deployments. It includes a parent `Application` resource that deploys other child applications like `nginx-ingress`.

---

## 📁 Repo Structure

```
argo-deployments/
├── Applications/
│   └── nginx-ingress.yaml        # ArgoCD Application for nginx
├── charts/
│   └── nginx/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
└── app-of-apps.yaml              # The root "App of Apps" Application
```

---

## ✅ Step-by-step Setup

### 1.Add repo to argocd & Root Application (`app-of-apps.yaml`)

# Go to your argo ui and add the repo

then 

This Application watches the `Applications/` folder and auto-syncs any `Application` manifests it finds.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  project: default
  source:
    repoURL: git@github.com:Kibria7533/argo-deployments.git
    targetRevision: HEAD
    path: Applications
    directory:
      recurse: true
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:
```bash
kubectl apply -f app-of-apps.yaml -n argocd
```

---

### 2. Child Application (`nginx-ingress.yaml`)

This is a Helm-based application pointing to your chart in `charts/nginx/`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
  namespace: argocd   # MUST match ArgoCD's namespace
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx-ingress   # Where nginx will be deployed
  project: default
  source:
    repoURL: git@github.com:Kibria7533/argo-deployments.git
    targetRevision: HEAD
    path: charts/nginx
    helm:
      valueFiles:
        - values.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Place this YAML inside `Applications/nginx-ingress.yaml`.

---

## 🚨 Common Mistakes & Fixes

| Issue | Fix |
|------|-----|
| `The project 'default' does not exist` | Ensure `default` project exists under ArgoCD Projects |
| Child app not showing up | Confirm it's in `Applications/` folder and `metadata.namespace` is `argocd` |
| Chart not rendering | Run `helm template charts/nginx` locally to check |
| App stuck in `Missing` or `Unknown` | Check ArgoCD has repo access and chart paths are correct |

---

## 🛠 Tips

- All Applications must be created in the **`argocd` namespace**.
- Use `CreateNamespace=true` in sync options if the target namespace doesn't exist.
- Keep your chart paths and `valueFiles` consistent with repo structure.

---

## 📬 Need Help?

Feel free to open an issue or contact the repo maintainer.