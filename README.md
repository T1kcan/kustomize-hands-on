# Kustomize Hands-on Lab: Multi-Environment Uygulama

Bu çalışma ile Kustomize kullanarak Kubernetes kaynaklarını `base`, `dev`, `prod` ortamları için nasıl özelleştireceğini öğreneceksin. Ayrıca `namespace`, `namePrefix`, `patch`, `configMap`, `secret`, `transformer` gibi ileri düzey özellikleri de pratik edeceğiz.

---

## Klasör Yapısı

```bash
my-kustomize-app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── labels-transformer.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patch.yaml
```
## base/ Manifests
- base/deployment.yaml
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
- base/service.yaml
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
- base/labels-transformer.yaml
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
- base/kustomization.yaml
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
- overlays/dev/
- overlays/dev/kustomization.yaml
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
- overlays/dev/patch.yaml
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
- overlays/prod/
- overlays/prod/kustomization.yaml
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
- overlays/prod/patch.yaml
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
## Uygulama
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
## Sonuç Özeti
Ortam	Namespace	Image	Replika	Prefix

dev	dev-namespace	nginx:1.21	2	dev-

prod	prod-namespace	nginx:1.25	5	prod-
