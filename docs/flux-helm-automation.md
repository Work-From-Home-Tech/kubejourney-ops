# Flux Automation using kube-prometheus-stack

The process involves converting the prometheus stack to GitOps deployment with flux.

Files 

```
monitoring
├── configs
│   ├── base
│   │   └── monitoring
│   │       └── namespace.yaml
│   ├── production
│   └── staging
└── controllers
    ├── base
    │   └── kube-prometheus-stack
    │       ├── kustomization.yaml
    │       ├── namespace.yaml
    │       ├── release.yaml
    │       └── repository.yaml
    ├── production
    └── staging
        ├── kube-prometheus-stack
        │   └── kustomization.yaml
        └── kustomization.yaml
```

1) Create a blank namespace file

```bash
kubectl create namespace monitoring --dry-run=client -o yaml > namespace.yaml
```

Edit the namespace.yaml file to match the following:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

2) Create the repository.yaml file using flux

```bash
flux create source helm kube-prometheus-stack \
--url=https://prometheus-community.github.io/helm-charts \
--namespace=monitoring \
--interval=24h \
--export > repository.yaml
```

This should create the following yaml file
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 24h
  url: https://prometheus-community.github.io/helm-charts
```

3) Create the release file

```bash
flux create helmrelease kube-prometheus-stack \
  --source=HelmRepository/kube-prometheus-stack \
  --chart=kube-prometheus-stack \
  --namespace=monitoring \
  --interval=24h \
  --export > release.yaml
```

Update the file to contain the following:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: "72.*"

      sourceRef:
        kind: HelmRepository
        name: kube-prometheus-stack
        namespace: monitoring
      interval: 12h
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  driftDetection:
    mode: enabled
    ignore:
      # Ignore "validated" annotation which is not inserted durings
      - paths: ["/metadata/annotations/prometheus-operator-validated"]
        target:
          kind: PrometheusRule
  values:
    grafana:
      adminPassword: wendell
```