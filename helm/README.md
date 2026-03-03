# Carbide REST Helm Charts

Helm charts for deploying the Carbide REST API platform services.

## Charts

| Chart | Path | Description |
|-------|------|-------------|
| `carbide-rest` | `charts/carbide-rest/` | Umbrella chart (api + workflow + site-manager + db) |
| `carbide-rest-site-agent` | `charts/carbide-rest-site-agent/` | Elektra site agent (deployed independently per-site) |

### Umbrella Sub-Charts

| Sub-Chart | Description |
|-----------|-------------|
| `carbide-rest-api` | REST API server (port 8388) |
| `carbide-rest-workflow` | Temporal workers (cloud-worker + site-worker) |
| `carbide-rest-site-manager` | Site lifecycle manager (TLS on port 8100) |
| `carbide-rest-db` | Database migration job (Bun ORM, idempotent) |

## Prerequisites

The following must be running before installing charts:

- **PostgreSQL** database
- **Temporal** server with `cloud` and `site` namespaces
- **Keycloak** authentication server
- **cert-manager** with ClusterIssuer `carbide-rest-ca-issuer`
- **Secrets**: `db-creds`, `keycloak-client-secret`, `temporal-encryption-key`, `temporal-client-cloud-certs`

> The Site CRD (`sites.forge.nvidia.io`) is bundled in `carbide-rest-site-manager/crds/` and installed automatically by Helm.

## Install

### Umbrella Chart (cloud-side services)

```bash
REPO=nvcr.io/0837451325059433/carbide-dev
TAG=latest
NS=carbide-rest

helm upgrade --install carbide-rest charts/carbide-rest/ \
  --namespace $NS --create-namespace \
  --set global.image.repository=$REPO \
  --set global.image.tag=$TAG
```

### Site Agent (deployed separately per-site)

Site agent requires a registered site (UUID + OTP). The chart must be installed first, then bootstrapped:

```bash
# 1. Install chart
helm upgrade --install carbide-rest-site-agent charts/carbide-rest-site-agent/ \
  --namespace $NS \
  --set global.image.repository=$REPO \
  --set global.image.tag=$TAG || true

# 2. Bootstrap site registration (creates site via API, patches ConfigMap/Secret)
./scripts/setup-local.sh site-agent

# 3. Site agent will stabilize after bootstrap
kubectl -n $NS rollout status statefulset/carbide-rest-site-agent --timeout=120s
```

## Uninstall

```bash
helm uninstall carbide-rest-site-agent -n carbide-rest
helm uninstall carbide-rest -n carbide-rest
```

## Configuration

### Umbrella Chart (`carbide-rest`)

Global values are passed to all sub-charts:

```yaml
global:
  image:
    repository: nvcr.io/0837451325059433/carbide-dev
    tag: "1.0.5"
    pullPolicy: IfNotPresent
  imagePullSecrets:
    - name: image-pull-secret
  certificate:
    issuerRef:
      kind: ClusterIssuer
      name: carbide-rest-ca-issuer
      group: cert-manager.io
```

### Site Agent Chart (`carbide-rest-site-agent`)

Standalone chart with its own `global` section (same structure as above).

See each chart's `values.yaml` for full configuration options.
