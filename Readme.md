# kustomize_multi_env

Multi-environment Kubernetes deployment with [Kustomize](https://kustomize.io/).
A shared **base** holds the common manifests, and per-environment **overlays**
(`dev`, `staging`, `prod`) patch only what differs: namespace, replicas,
resources, image tags and config.

## Project Structure

```
kustomize_multi_env/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── index.html
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   ├── deployment-patch.yaml
    │   └── configmap.env
    ├── staging/   (same files as dev)
    └── prod/      (same files as dev)
```

| Path              | Purpose                                             |
|-------------------|-----------------------------------------------------|
| `base/`           | Environment-agnostic manifests shared by all envs   |
| `overlays/<env>/` | Environment-specific customization on top of `base` |

## Prerequisites

- A Kubernetes cluster (minikube, kind, EKS, GKE, AKS, ...)
- `kubectl` v1.14+ (Kustomize built in via `-k`)
- Optional: standalone `kustomize` CLI (`brew install kustomize`)

```bash
kubectl version --client
kustomize version
kubectl config current-context
```

## How It Works

1. `base/` defines the app once (Deployment, Service, static `index.html`).
2. Each overlay references the base via `resources: - ../../base`.
3. Overlays add patches, generators, a namespace, labels and image overrides.
4. `kustomize build overlays/<env>` renders final YAML, no templating needed.

| Setting   | dev | staging | prod |
|-----------|-----|---------|------|
| Namespace | dev | staging | prod |
| Replicas  | 1   | 3       | 5    |
| Resources | low | medium  | high |

## Quick Start

```bash
git clone https://github.com/1-DARK/kustomize_multi_env.git
cd kustomize_multi_env

kubectl kustomize overlays/dev     # preview
kubectl apply -k overlays/dev      # deploy
kubectl get all -n dev             # verify
```

## Commands

### Build / preview

```bash
kustomize build base
kustomize build overlays/prod
kubectl kustomize overlays/prod
kustomize build overlays/prod > prod-rendered.yaml
```

### Deploy

```bash
kubectl apply -k overlays/dev
kubectl apply -k overlays/staging
kubectl apply -k overlays/prod

kustomize build overlays/dev | kubectl apply -f -   # via CLI
kubectl apply -k overlays/dev --dry-run=server      # dry run
kubectl diff -k overlays/prod                       # diff first
```

### Inspect

```bash
kubectl get all -n dev
kubectl describe deployment <name> -n prod
kubectl logs -f deployment/<name> -n prod
kubectl get events -n dev --sort-by=.lastTimestamp
kubectl port-forward svc/<service-name> 8080:80 -n dev
```

### Rollout / rollback

```bash
kubectl rollout status deployment/<name> -n prod
kubectl rollout history deployment/<name> -n prod
kubectl rollout undo deployment/<name> -n prod
```

### Edit an overlay from the CLI

```bash
cd overlays/dev
kustomize edit set image nginx=nginx:1.27
kustomize edit set namespace dev
kustomize edit add patch --path deployment-patch.yaml
kustomize edit fix    # migrate deprecated fields
```

### Delete

```bash
kubectl delete -k overlays/dev
```

## Configuration Examples

`overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - ../../base
  - namespace.yaml

patches:
  - path: deployment-patch.yaml

configMapGenerator:
  - name: app-config
    envs:
      - configmap.env

images:
  - name: nginx
    newTag: "1.27"
```

`overlays/prod/deployment-patch.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: app
          resources:
            requests: { cpu: "500m", memory: "256Mi" }
            limits:   { cpu: "1000m", memory: "512Mi" }
```

## Adding a New Environment

```bash
cp -r overlays/staging overlays/qa
sed -i 's/staging/qa/g' overlays/qa/kustomization.yaml overlays/qa/namespace.yaml
kubectl apply -k overlays/qa
```

## Troubleshooting

| Problem                               | Fix                                                        |
|---------------------------------------|------------------------------------------------------------|
| `kustomization.yaml` not found        | Run against a directory that contains it                   |
| `namespace "dev" not found`           | Add `namespace.yaml` to `resources` or `kubectl create ns` |
| `patchesStrategicMerge is deprecated` | Use `patches:` or run `kustomize edit fix`                 |
| Patch not applied                     | `metadata.name` and kind must match the base exactly       |
| `-k` differs from `kustomize build`   | kubectl bundles an older kustomize; pipe the CLI output    |

## Best Practices

- Keep `base/` free of environment-specific values.
- Run `kubectl diff -k` before every prod apply.
- Pin image tags (avoid `latest`) and promote the same tag across envs.
- Never commit real secrets; use Sealed Secrets, External Secrets or SOPS.