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
argocd-application.yaml                          # the ArgoCD Application resource itself
helm/opentelemetry-demo/
├── values.yaml                                    # image/tag/replica/resources per service - the only thing CI touches
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

Point ArgoCD at this repo by applying the Application manifest:

```bash
kubectl apply -f argocd-application.yaml
```

`destination.server: https://kubernetes.default.svc` targets whatever cluster ArgoCD itself is running on — currently minikube. Everything else (namespace creation, sync, pruning, self-healing) is handled by the `syncPolicy.automated` block.

### Notes on values.yaml

- `frontendProxy.ingress.enabled` is `false` by default: the ingress uses the AWS ALB ingress class, which only resolves on EKS with the AWS Load Balancer Controller installed. Left enabled on minikube, ArgoCD would report the Application stuck `Progressing` forever waiting for an address that will never come.
- A few services still point at the public upstream `ghcr.io/open-telemetry/demo` image rather than an `amirweiser`-built one — noted inline in `values.yaml`. CI in `demo-src` is fully capable of building them; those specific tags just haven't been promoted yet. Swapping them over is a `values.yaml` edit, not a pipeline change.
