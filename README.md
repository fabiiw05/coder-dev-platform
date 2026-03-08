# coder-dev-platform

[日本語](./README.ja.md)

Kubernetes-based developer platform powered by [Coder](https://coder.com/), with [CloudNativePG](https://cloudnative-pg.io/) for PostgreSQL and [Tailscale](https://tailscale.com/) for secure network access. All components are deployed via [Helmfile](https://helmfile.readthedocs.io/).

## Architecture

| Component | Chart | Purpose |
|-----------|-------|---------|
| Tailscale Operator | `tailscale/tailscale-operator` | Secure network access and Ingress via Tailscale |
| CloudNativePG Operator | `cnpg/cloudnative-pg` | PostgreSQL operator for Kubernetes |
| PostgreSQL Cluster | `cnpg/cluster` | Coder's database (managed by CloudNativePG) |
| Coder | `coder-v2/coder` | Cloud development platform |

### Deployment order

Helmfile `needs` ensures correct ordering:

```
tailscale-operator  (independent)
cnpg operator  -->  coder-db (PostgreSQL)  -->  coder
```

## Prerequisites

- [mise](https://mise.jdx.dev/) -- manages CLI tool versions (helm, helmfile, kubectl)
- Access to a Kubernetes cluster (`kubectl` context configured)
- Tailscale OAuth client credentials (`client_id` and `client_secret`)

## Setup

### 1. Install CLI tools

```bash
mise install
```

This installs the exact versions defined in `mise.toml`:

| Tool | Version |
|------|---------|
| helm | 4.1.1 |
| helmfile | 1.3.0 |
| kubectl | 1.35.1 |

### 2. Create the Tailscale namespace and OAuth secret

```bash
kubectl create ns tailscale

kubectl create secret generic operator-oauth \
  --from-literal=client_id=<YOUR_CLIENT_ID> \
  --from-literal=client_secret=<YOUR_CLIENT_SECRET> \
  -n tailscale
```

### 3. Deploy all components

```bash
helmfile apply
```

This deploys (in order): Tailscale operator, CloudNativePG operator, PostgreSQL cluster, and Coder.

### Other useful commands

```bash
# Preview changes before applying
helmfile diff

# Render templates locally (dry-run)
helmfile template

# Lint the chart values
helmfile lint

# Tear down all releases
helmfile destroy
```

## Configuration

Values files are organized under `values/`:

| File | Component | Description |
|------|-----------|-------------|
| `values/tailscale.yaml` | Tailscale Operator | OAuth config, operator tags, proxy settings, API server proxy |
| `values/cnpg-operator.yaml` | CloudNativePG Operator | Operator replica count, monitoring |
| `values/cnpg-cluster.yaml` | PostgreSQL Cluster | Instance count, storage, PostgreSQL version, database/user |
| `values/coder.yaml` | Coder | DB connection, access URL, Ingress (Tailscale), OAuth |

### Key settings

#### Coder (`values/coder.yaml`)

| Setting | Description |
|---------|-------------|
| `coder.env` `CODER_PG_CONNECTION_URL` | Auto-populated from CloudNativePG secret (`coder-db-app`) |
| `coder.env` `CODER_ACCESS_URL` | URL for workspace connections (`https://coder`) |
| `coder.ingress.className` | Set to `tailscale` for Tailscale Ingress |

#### PostgreSQL (`values/cnpg-cluster.yaml`)

| Setting | Description |
|---------|-------------|
| `cluster.instances` | Number of PostgreSQL replicas (default: `1`) |
| `cluster.storage.size` | Persistent volume size (default: `10Gi`) |
| `databases.coder.owner` | Database owner user (default: `coder`) |

#### Tailscale (`values/tailscale.yaml`)

| Setting | Description |
|---------|-------------|
| `oauth.clientSecret` | Name of the Kubernetes secret containing OAuth credentials |
| `operatorConfig.defaultTags` | ACL tags assigned to the operator (default: `tag:k8s-operator`) |
| `proxyConfig.defaultTags` | ACL tags assigned to proxies (default: `tag:k8s`) |
| `apiServerProxyConfig.mode` | API server proxy mode (`true`, `false`, `noauth`) |

## Accessing Coder

Coder is exposed via Tailscale Ingress. After deployment:

1. Ensure [HTTPS certificates are enabled](https://tailscale.com/docs/how-to/set-up-https-certificates) on your Tailnet
2. Access Coder at `https://coder` (via Tailscale MagicDNS)
3. Create your first admin user through the web UI

## References

- [Coder Docs -- Kubernetes Install](https://coder.com/docs/install/kubernetes)
- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator)
- [Setting up HTTPS Certificates](https://tailscale.com/docs/how-to/set-up-https-certificates)
- [Helmfile Documentation](https://helmfile.readthedocs.io/)
- [mise Documentation](https://mise.jdx.dev/)
