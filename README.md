# Install helm
```sh
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
#  GitHub Actions: Full EKS Bootstrap with ArgoCD, Ingress, and Cert-Manager

This GitHub Actions workflow performs the following tasks:

- Accesses an **Amazon EKS** cluster
- Installs **ArgoCD** (if not installed)
- Installs **Cert-Manager**
- Installs **NGINX Ingress Controller**
- Creates an **Ingress** with SSL to expose the ArgoCD dashboard
- Uses a DNS name from an environment variable (`ARGO_DOMAIN`)

and on env (deploy-argocd) update those values

env:
  AWS_REGION: us-east-1
  CLUSTER_NAME: your-cluster-name

---

## 🔐 Required GitHub Secrets

| Secret Name            | Description                          |
|------------------------|--------------------------------------|
| `AWS_ACCESS_KEY_ID`    | AWS access key ID                    |
| `AWS_SECRET_ACCESS_KEY`| AWS secret access key                |
| `ARGO_DOMAIN`          | DNS name for ArgoCD (e.g. `argocd.example.com`) |

---

## 🛠️ Workflow File

Save this as `.github/workflows/eks-bootstrap.yml`:

```yaml
name: EKS Bootstrap with ArgoCD

on:
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  CLUSTER_NAME: <your-cluster-name>

jobs:
  bootstrap:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Install kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: latest

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

      - name: Install Cert-Manager
        run: |
          kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
          kubectl rollout status deployment/cert-manager -n cert-manager --timeout=2m

      - name: Install NGINX Ingress Controller
        run: |
          kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
          kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx --timeout=2m

      - name: Check if ArgoCD is already installed
        id: check-argocd
        run: |
          if kubectl get namespace argocd > /dev/null 2>&1; then
            echo "ArgoCD already installed"
            echo "exists=true" >> $GITHUB_OUTPUT
          else
            echo "ArgoCD not installed"
            echo "exists=false" >> $GITHUB_OUTPUT
          fi

      - name: Install ArgoCD
        if: steps.check-argocd.outputs.exists == 'false'
        run: |
          kubectl create namespace argocd
          kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
          kubectl rollout status deployment/argocd-server -n argocd --timeout=2m

      - name: Create ArgoCD Ingress with SSL
        run: |
          cat <<EOF | kubectl apply -f -
          apiVersion: networking.k8s.io/v1
          kind: Ingress
          metadata:
            name: argocd-server
            namespace: argocd
            annotations:
              nginx.ingress.kubernetes.io/ssl-redirect: "true"
              nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
              cert-manager.io/cluster-issuer: "letsencrypt-prod"
          spec:
            ingressClassName: nginx
            tls:
              - hosts:
                  - ${{ secrets.ARGO_DOMAIN }}
                secretName: argocd-tls
            rules:
              - host: ${{ secrets.ARGO_DOMAIN }}
                http:
                  paths:
                    - path: /
                      pathType: Prefix
                      backend:
                        service:
                          name: argocd-server
                          port:
                            number: 443
          EOF

      - name: Create ClusterIssuer for Cert-Manager
        run: |
          cat <<EOF | kubectl apply -f -
          apiVersion: cert-manager.io/v1
          kind: ClusterIssuer
          metadata:
            name: letsencrypt-prod
          spec:
            acme:
              server: https://acme-v02.api.letsencrypt.org/directory
              email: your-email@example.com
              privateKeySecretRef:
                name: letsencrypt-prod
              solvers:
                - http01:
                    ingress:
                      class: nginx
          EOF
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