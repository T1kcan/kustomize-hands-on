# Kustomize Hands-on Lab: Multi-Environment Uygulama

Bu çalışma ile Kustomize kullanarak Kubernetes kaynaklarını `base`, `dev`, `prod` ortamları için nasıl özelleştireceğini öğreneceksin. Ayrıca `namespace`, `namePrefix`, `patch`, `configMap`, `secret`, `transformer` gibi ileri düzey özellikleri de pratik edeceğiz.

---

## Klasör Yapısı

```bash
.
|-- README.md
|-- argocd
|   |-- argocd-dev-app.yaml
|   `-- argocd-prod-app.yaml
|-- base
|   |-- deployment.yaml
|   |-- kustomization.yaml
|   |-- labels-transformer.yaml
|   `-- service.yaml
`-- overlays
    |-- dev
    |   |-- kustomization.yaml
    |   `-- patch.yaml
    `-- prod
        |-- kustomization.yaml
        `-- patch.yaml
```
## base/ Manifests
1. base/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:latest
        ports:
        - containerPort: 80
```
2. base/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
3. base/labels-transformer.yaml
```yaml
apiVersion: builtin
kind: LabelTransformer
metadata:
  name: add-common-labels
labels:
  managed-by: kustomize
  team: platform
fieldSpecs:
  - path: metadata/labels
    create: true
```yaml
4. base/kustomization.yaml
```yaml
resources:
  - deployment.yaml
  - service.yaml

transformers:
  - labels-transformer.yaml

commonLabels:
  app.kubernetes.io/part-of: my-app

configMapGenerator:
  - name: app-config
    literals:
      - ENV=default
      - LOG_LEVEL=info

secretGenerator:
  - name: app-secret
    literals:
      - PASSWORD=changeme
```
## overlays/dev/ Manifests
1. overlays/dev/kustomization.yaml
```yaml
resources:
  - ../../base

namePrefix: dev-

namespace: dev-namespace

patchesStrategicMerge:
  - patch.yaml

commonLabels:
  environment: dev
```
2. overlays/dev/patch.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: my-app
        image: nginx:1.21
```
## overlays/prod/
1. overlays/prod/kustomization.yaml
```yaml
resources:
  - ../../base

namePrefix: prod-

namespace: prod-namespace

patchesStrategicMerge:
  - patch.yaml

commonLabels:
  environment: prod
```
2. overlays/prod/patch.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: my-app
        image: nginx:1.25
```
## Argocd Manifests
1. argocd-dev-app.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/T1kcan/kustomize-hands-on.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
2. argocd-prod-app.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/T1kcan/kustomize-hands-on.git
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
## Uygulama
Manifestler direkt çalıştırılabilir ya da Gitops aracılığıyla deploy edilebilir.
### Direkt 
Namespace Oluştur
```bash
kubectl create namespace dev-namespace
kubectl create namespace prod-namespace
```
Deploy Et
```bash
Copy code
kubectl apply -k overlays/dev/
kubectl apply -k overlays/prod/
```
Durum Kontrolü
```bash
kubectl get all -n dev-namespace
kubectl get all -n prod-namespace
```
## Temizlik
```bash
kubectl delete -k overlays/dev/
kubectl delete -k overlays/prod/
```
### Gitops
ArgoCD:
```bash
kubectl apply -f argocd-dev-app.yaml
kubectl apply -f argocd-prod-app.yaml
```
ArgoCD CLI:
```bash
argocd login localhost:8080
argocd app create my-app-dev \
  --repo https://github.com/T1kcan/kustomize-hands-on \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev-namespace \
  --sync-policy automated
```
Test:
```bash
cd overlays/dev
vi patch.yaml
# image: nginx:1.24
git commit -am "Update dev image"
git push
```
## Sonuç Özeti
Ortam	Namespace	Image	Replika	Prefix

dev	dev-namespace	nginx:1.21	2	dev-

prod	prod-namespace	nginx:1.25	5	prod-
