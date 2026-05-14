---
layout: default
title: Kubernetes Deployment
nav_order: 8
---

# Kubernetes Deployment

This page covers deploying the onisin backend services (`oosai`, `oosgql`, NATS)
to a Kubernetes cluster. The desktop clients (`oos`, `oosd`) always run locally
and connect to the cluster over the network.

## Architecture

```
Mac (local)
├── oos   ───────────────────────────┬─────── nats://cluster-ip:4222
└── oosd  ───────────────────────────┘

Kubernetes (namespace: onisin)
├── NATS         :4222  ←─ Traefik IngressRouteTCP (TCP passthrough)
├── oosgql       :4000  ←─ Traefik IngressRoute /gql
└── oosai        :4100  ←─ Traefik IngressRoute /ai
```

`oos` and `oosd` communicate exclusively over NATS — `oosgql` and `oosai`
subscribe to NATS subjects and never need to be reached directly by the
desktop clients.

## Prerequisites

- A running Kubernetes cluster (see [Local development](#local-development))
- `kubectl` configured for that cluster
- `helm` ≥ 3
- Docker (or compatible) for building images
- A PostgreSQL database reachable from inside the cluster
  (see [Database](#database))

## Local development

The recommended local setup is **OrbStack** with its built-in Kubernetes.
OrbStack provides a working `LoadBalancer` without any additional configuration,
which means NATS and HTTP are reachable from your Mac immediately after
deployment.

```bash
# Install OrbStack (if not already installed)
brew install --cask orbstack

# Enable Kubernetes in OrbStack → Settings → Kubernetes

# Verify context
kubectl config use-context orbstack
```

Other options such as Colima + k3s also work, but require additional
port-forwarding configuration because Colima’s SSH-based port forwarder
does not automatically expose `LoadBalancer` services.

## Database

For production, use a managed PostgreSQL service (AWS RDS, Google CloudSQL,
etc.) or a dedicated database host. **Do not run PostgreSQL as a Kubernetes
`Deployment` for production workloads** — data persistence requires careful
`PersistentVolume` configuration and backup strategies that are out of scope
for this guide.

For local development with OrbStack or Docker Desktop, PostgreSQL running on
your Mac is reachable from inside the cluster at the special hostname
`host.docker.internal`:

```
postgres://postgres:demo@host.docker.internal:5432/onisin
```

> **Note:** `host.docker.internal` works with OrbStack and Docker Desktop.
> It is not available in production clusters. Use the real database hostname
> there.

## Ingress

The Helm chart and the raw manifests include Traefik-specific ingress resources
(CRDs `IngressRouteTCP` and `IngressRoute`). Traefik requires a custom TCP
EntryPoint for NATS — see `deploy/traefik-values.yaml` for the required Helm
values.

If you use a different ingress controller, set `ingress.traefik.enabled=false`
in your values file and configure your own ingress:

| Service | Port | Protocol | Path prefix |
|---------|------|----------|-------------|
| nats    | 4222 | TCP      | — (passthrough) |
| oosgql  | 4000 | HTTP     | `/gql` (strip prefix before forwarding) |
| oosai   | 4100 | HTTP     | `/ai` (strip prefix before forwarding) |

## Known Kubernetes gotchas

### Service env-var injection conflict

Kubernetes automatically injects `<SERVICENAME>_PORT=tcp://ip:port` into every
container in the namespace for each Service that exists. Because the services
are named `oosai` and `oosgql`, k8s injects `OOSAI_PORT` and `OOSGQL_PORT` —
which overwrites the numeric port the application expects.

The Helm chart and the raw manifests already handle this by declaring explicit
`OOSAI_PORT: "4100"` and `OOSGQL_PORT: "4000"` env vars in the Deployments.
If you rename the Services you do not need this workaround.

### Listen address must be 0.0.0.0

Both services default to binding on `localhost`. In Kubernetes, readiness and
liveness probes connect to the Pod IP, not `localhost`, so the probe will always
fail unless the service binds on `0.0.0.0`. The Deployments set `OOSAI_HOST`
and `OOSGQL_HOST` to `0.0.0.0` for this reason.

## Deploying with Helm (recommended)

### Install Traefik

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik \
  --namespace kube-system \
  -f deploy/traefik-values.yaml
```

`deploy/traefik-values.yaml` defines the `nats` TCP EntryPoint on port 4222
alongside the standard `web` (80) and `websecure` (443) EntryPoints.

### Create the namespace

```bash
kubectl create namespace onisin
```

### Install the chart

```bash
helm upgrade --install onisin deploy/helm/onisin \
  --namespace onisin \
  --set db.url='postgres://user:password@db-host:5432/onisin'
```

For local development with OrbStack:

```bash
helm upgrade --install onisin deploy/helm/onisin \
  --namespace onisin \
  --set db.url='postgres://postgres:demo@host.docker.internal:5432/onisin'
```

Override the embedding endpoint if you are not running Ollama on the default
address:

```bash
  --set embed.url='http://my-ollama-host:11434/v1' \
  --set embed.model='granite-embedding:latest'
```

To disable Traefik ingress and bring your own:

```bash
  --set ingress.traefik.enabled=false
```

### Verify

```bash
kubectl get pods -n onisin
# All pods should show READY 1/1.

# Health checks (replace IP with your LoadBalancer external IP)
export LB=$(kubectl get svc traefik -n kube-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$LB/gql/health   # {"status":"ok","dialect":"postgres"}
curl http://$LB/ai/health    # {"status":"ok","embedModel":"..."}
```

NATS is reachable at `nats://$LB:4222`. Enter this URL in `oos` and `oosd`
under Settings → NATS URL.

## Deploying with raw manifests

If you prefer plain `kubectl apply` without Helm:

```bash
# 1. Traefik
helm upgrade --install traefik traefik/traefik \
  --namespace kube-system \
  -f deploy/traefik-values.yaml

# 2. Namespace
kubectl apply -f deploy/namespace.yaml

# 3. Secret (edit the file first)
cp deploy/secrets.yaml.example deploy/secrets.yaml
# Edit deploy/secrets.yaml and fill in PG_URL
kubectl apply -f deploy/secrets.yaml

# 4. Everything else
kubectl apply -f deploy/config.yaml
kubectl apply -f deploy/nats.yaml
kubectl apply -f deploy/oosgql.yaml
kubectl apply -f deploy/oosai.yaml
kubectl apply -f deploy/ingress.yaml
```

## Building images

Images are built with Docker and tagged `onisin/oosai:dev` /
`onisin/oosgql:dev`. Run from the repository root:

```bash
# arm64 (M-series Mac, OrbStack default)
docker build -t onisin/oosai:dev  -f apps/oosai/Dockerfile  .
docker build -t onisin/oosgql:dev -f apps/oosgql/Dockerfile .

# x64 (CI, Linux servers)
docker build --build-arg BUN_TARGET=bun-linux-x64 \
  -t onisin/oosai:dev  -f apps/oosai/Dockerfile  .
docker build --build-arg BUN_TARGET=bun-linux-x64 \
  -t onisin/oosgql:dev -f apps/oosgql/Dockerfile .
```

With OrbStack as the active Docker context, images built this way are
automatically available inside the cluster without a registry push.
