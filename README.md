# opentelemetry-demo-gitops

The Helm chart and ArgoCD Application that deploy the [OpenTelemetry demo](https://github.com/AmirWeiser/opentelemetry-demo-src) microservices app — 19 application services plus a minimal observability stack — entirely through GitOps. Nothing here gets `kubectl apply`'d by hand; ArgoCD watches this repo and reconciles the cluster to match it.

This is one of three repositories that make up the full project:

| Repo | Role |
|---|---|
| [opentelemetry-aws-infra](https://github.com/AmirWeiser/opentelemetry-aws-infra) | The infrastructure the app runs on |
| **opentelemetry-demo-gitops** (this repo) | The Helm chart ArgoCD deploys |
| [opentelemetry-demo-src](https://github.com/AmirWeiser/opentelemetry-demo-src) | Application source + CI pipeline |

## How it fits together

```
demo-src (app code)  --CI builds & pushes-->  GHCR
       |                                         |
       '---- CI bumps image tag in ------->  values.yaml (this repo)
                                                  |
                                          ArgoCD auto-sync
                                                  |
                                             Kubernetes
```

CI in `opentelemetry-demo-src` is the only thing that ever writes to this repo — every commit to `values.yaml` bumping an image tag is machine-generated (`chore: bump image tag(s) to <sha>`), triggered by a successful build in the source repo. ArgoCD's `automated` sync policy (`prune: true`, `selfHeal: true`) picks up the new commit and reconciles the cluster within its polling interval, no manual `helm upgrade` involved.

## Repo layout

```
argocd-application.yaml                          # ArgoCD Application - minikube (values.yaml only)
argocd-application-eks.yaml                        # ArgoCD Application - EKS (values.yaml + values-eks.yaml)
helm/opentelemetry-demo/
├── values.yaml                                    # image/tag/replica/resources per service - the only thing CI touches
├── values-eks.yaml                                 # EKS-only overrides layered on top (ingress, pull secret)
├── Chart.yaml
└── templates/
    ├── <service>/{deploy,svc}.yaml                 # one directory per service, 19 total
    ├── frontendproxy/ingress.yaml                   # ALB ingress, gated off by default (see below)
    ├── otelcollector/{deploy,svc,configmap}.yaml    # OTel Collector - receives OTLP from every service
    ├── jaeger/{deploy,svc}.yaml                     # trace backend
    ├── prometheus/{deploy,svc,configmap}.yaml       # metrics backend, scrapes the collector
    └── grafana/{deploy,svc,configmap*}.yaml         # dashboards, provisioned via ConfigMap
```

Every service directory follows the same shape: a `Deployment` with startup/readiness/liveness probes and CPU/memory requests+limits, and a `Service`. What varies between services (image, tag, replica count, resource sizing) lives in `values.yaml`; everything structural (env wiring, ports, probe config, security context) is stable and stays in the templates.

## The observability stack

Every application service already ships OTLP traces/metrics/logs to `OTEL_EXPORTER_OTLP_ENDPOINT`. This chart deploys the pieces that receive and expose that data:

- **OTel Collector** — receives OTLP over gRPC/HTTP, fans out to the backends below.
- **Jaeger** — trace storage and UI (in-memory, no external DB — fine for a demo, not for production retention).
- **Prometheus** — scrapes metrics off the collector's Prometheus exporter.
- **Grafana** — dashboards, provisioned entirely via ConfigMap (datasources + a pre-built "OpenTelemetry Demo - Overview" dashboard). No manual click-ops setup after deploy.

All four run single-replica with deliberately small resource footprints — this chart is routinely exercised on a 4-core laptop via minikube, not a beefy cluster. Grafana's image is pinned to `10.4.2` rather than `latest` for the same reason: `latest`'s heavier internal apiserver/unified-storage stack was slow enough on constrained CPU that its first-boot SQLite migration didn't reliably finish inside a 5-minute startup probe window.

## Deploying

Two ArgoCD Application manifests, one per target - both deploy the same name (`opentelemetry-demo` in the `argocd` namespace), so only ever apply one to a given cluster:

```bash
# minikube - loads values.yaml only
kubectl apply -f argocd-application.yaml

# EKS - loads values.yaml, then layers values-eks.yaml on top
kubectl apply -f argocd-application-eks.yaml
```

`destination.server: https://kubernetes.default.svc` targets whatever cluster ArgoCD itself is running on in both cases - the only difference between the two manifests is `spec.source.helm.valueFiles`. Everything else (namespace creation, sync, pruning, self-healing) is handled by the `syncPolicy.automated` block.

### Why two values files instead of one

`values.yaml`'s own defaults are minikube-safe (`frontendProxy.ingress.enabled: false`, `global.imagePullSecrets: []`) - flipping them for EKS used to mean hand-editing `values.yaml` and reverting it every time the target environment changed. `values-eks.yaml` holds just the two settings that are only ever correct on one environment or the other:

- `frontendProxy.ingress.enabled: true` - the ALB ingress class needs the [AWS Load Balancer Controller](https://github.com/AmirWeiser/opentelemetry-aws-infra) installed, which only exists on EKS. Left enabled on minikube, ArgoCD would report the Application stuck `Progressing` forever waiting for an address that never comes. The Ingress rule itself has no `host:` filter, so the ALB's own DNS name works directly in a browser without owning a domain.
- `global.imagePullSecrets: [{name: ghcr-pull-secret}]` - references a Secret by name only; the credential itself is never committed here (a pull secret's contents are base64, not encryption). It's created directly in the cluster by `opentelemetry-aws-infra`'s bootstrap script from a locally-held PAT. Minikube doesn't need it and doesn't have that secret, so it stays out of the base `values.yaml`.

Every service now runs its own CI-built `ghcr.io/amirweiser/<service>` image - there's no longer a public-fallback exception list to track here.
