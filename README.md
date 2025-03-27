# Install helm
```sh
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
# Install Argocd (Node you have to make its type NodePort form kubernetes dashbord service)
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
# Get argocd secret and you know default username is admin

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
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

# Go to your argo ui and the repo

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