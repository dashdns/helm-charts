# DashDNS Helm Charts

Helm chart repository for the **DNS Mesh** stack — an eBPF-powered DNS proxy and Kubernetes-native policy controller.

[![Helm](https://img.shields.io/badge/helm-3.x-blue)](https://helm.sh)
[![License](https://img.shields.io/github/license/dashdns/helm-charts)](LICENSE)

## Charts

| Chart | Description | Version |
|-------|-------------|---------|
| [`dashdns`](#dashdns) | eBPF-based DNS proxy running as a DaemonSet (XDP/TC filtering) | 2.0.2 |
| [`dns-mesh-controller`](#dns-mesh-controller) | Kubernetes policy controller + admission webhook | 2.0.3 |
| [`dns-mesh-full`](#dns-mesh-full-umbrella) | Umbrella chart — installs the full stack in one release | 1.0.0 |

---

## Add the Helm Repository

```bash
helm repo add dashdns https://dashdns.github.io/helm-charts
helm repo update
```

---

## dns-mesh-full (Umbrella)

The recommended way to deploy the full stack. Installs both `dashdns` and `dns-mesh-controller` in a single Helm release.

### Quick install

```bash
helm install dns-mesh dashdns/dns-mesh-full \
  --namespace dns-mesh \
  --create-namespace
```

### Install with custom values

```bash
helm install dns-mesh dashdns/dns-mesh-full \
  --namespace dns-mesh \
  --create-namespace \
  --set dashdns.dns.interface=eth0 \
  --set dashdns.dns.upstream=8.8.8.8:53 \
  --set dns-mesh-controller.webhook.certChain.ca="<base64-ca>"
```

Or use a values file:

```bash
helm install dns-mesh dashdns/dns-mesh-full \
  --namespace dns-mesh \
  --create-namespace \
  -f my-values.yaml
```

### Selectively disable components

```bash
# Install only the controller (no DashDNS daemon)
helm install dns-mesh dashdns/dns-mesh-full \
  --set dashdns.enabled=false

# Install only DashDNS (no controller)
helm install dns-mesh dashdns/dns-mesh-full \
  --set dns-mesh-controller.enabled=false
```

### Upgrade

```bash
helm upgrade dns-mesh dashdns/dns-mesh-full \
  --namespace dns-mesh \
  -f my-values.yaml
```

### Uninstall

```bash
helm uninstall dns-mesh --namespace dns-mesh
```

### Values reference

All sub-chart values are namespaced under their chart name. Full defaults are in [`dns-mesh-full/values.yaml`](dns-mesh-full/values.yaml).

```yaml
dashdns:
  enabled: true
  dns:
    interface: eth0
    upstream: "1.1.1.1:53"
    ipBlocklistUrl: "http://dns-mesh-stack-service.default:5959/api/policies"
    ipBlocklistInterval: "5s"

dns-mesh-controller:
  enabled: true
  controller:
    service:
      apiPort: 5959
  webhook:
    certChain:
      ca: ""    # base64-encoded CA cert
      cert: ""  # base64-encoded TLS cert
      key: ""   # base64-encoded TLS key
```

---

## dashdns

eBPF-based DNS proxy that attaches XDP/TC programs to a network interface. Runs as a **DaemonSet** so every node intercepts and filters DNS traffic.

### Quick install

```bash
helm install dashdns dashdns/dashdns \
  --namespace dashdns \
  --create-namespace
```

### Key values

| Key | Default | Description |
|-----|---------|-------------|
| `image.repository` | `emirozbir/dnsd` | Container image |
| `image.tag` | `v2.0.3` | Image tag |
| `dns.interface` | `eth0` | Network interface for XDP/TC attachment |
| `dns.upstream` | `1.1.1.1:53` | Upstream DNS resolver |
| `dns.blocklist` | `""` | Comma-separated global domain blocklist |
| `dns.blockIps` | `""` | Comma-separated IPs to block in responses |
| `dns.blockedDns` | `""` | Comma-separated blocked DNS server IPs |
| `dns.ipBlocklist` | `""` | Per-IP blocklist (`IP:domain1,domain2;IP2:domain3`) |
| `dns.ipBlocklistUrl` | `http://dns-mesh-stack-service.default:5959/api/policies` | URL to fetch per-IP policy JSON |
| `dns.ipBlocklistInterval` | `5s` | Policy refresh interval |
| `service.type` | `NodePort` | Kubernetes service type |
| `service.nodePort` | `30053` | NodePort for DNS (UDP/53) |
| `prometheus.enabled` | `false` | Enable Prometheus ServiceMonitor |
| `securityContext.privileged` | `true` | Required for eBPF operations |

### Security context

DashDNS requires elevated privileges to load eBPF programs:

```yaml
securityContext:
  privileged: true
  capabilities:
    add:
      - SYS_ADMIN
      - NET_ADMIN
      - SYS_RESOURCE
```

### Enable Prometheus metrics

```bash
helm install dashdns dashdns/dashdns \
  --set prometheus.enabled=true \
  --set prometheus.serviceMonitorDiscoverLabels.release=prometheus-operator
```

---

## dns-mesh-controller

Kubernetes controller that manages `DNSPolicy` CRDs and exposes a policy API consumed by DashDNS. Also includes an **admission webhook** to annotate pods at creation time.

### Quick install

```bash
helm install dns-mesh-controller dashdns/dns-mesh-controller \
  --namespace dns-mesh-controller \
  --create-namespace
```

### Key values

| Key | Default | Description |
|-----|---------|-------------|
| `controller.image.repository` | `emirozbir/dashdns-controller` | Controller image |
| `controller.image.tag` | `v2.0.3` | Image tag |
| `controller.replicaCount` | `1` | Number of controller replicas |
| `controller.service.apiPort` | `5959` | Port for the policy REST API |
| `controller.service.port` | `8443` | Controller metrics/health port |
| `webhook.image.repository` | `emirozbir/dashdns-admission-webhook` | Webhook image |
| `webhook.image.tag` | `v2.0.3` | Image tag |
| `webhook.replicas` | `1` | Number of webhook replicas |
| `webhook.dns_service.name` | `dnsd-dashdns` | DashDNS service name the webhook references |
| `webhook.dns_service.namespace` | `default` | Namespace of the DashDNS service |
| `webhook.certChain.ca` | `""` | Base64-encoded CA certificate |
| `webhook.certChain.cert` | `""` | Base64-encoded TLS certificate |
| `webhook.certChain.key` | `""` | Base64-encoded TLS private key |

### Webhook TLS certificates

The admission webhook requires a valid TLS certificate. Pass the base64-encoded values:

```bash
helm install dns-mesh-controller dashdns/dns-mesh-controller \
  --set webhook.certChain.ca="$(cat ca.crt | base64)" \
  --set webhook.certChain.cert="$(cat tls.crt | base64)" \
  --set webhook.certChain.key="$(cat tls.key | base64)"
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Kubernetes Node                                        │
│                                                         │
│  ┌──────────────┐   DNS query    ┌──────────────────┐  │
│  │     Pod      │ ─────────────▶ │    DashDNS       │  │
│  │  (annotated) │                │  (eBPF DaemonSet) │  │
│  └──────────────┘                │                  │  │
│                                  │  XDP/TC filters  │  │
│                                  └────────┬─────────┘  │
│                                           │ policy poll │
└───────────────────────────────────────────┼─────────────┘
                                            │
                              ┌─────────────▼──────────────┐
                              │   dns-mesh-controller       │
                              │                             │
                              │  ┌───────────────────────┐ │
                              │  │  Policy API (:5959)    │ │
                              │  │  /api/policies         │ │
                              │  └───────────────────────┘ │
                              │  ┌───────────────────────┐ │
                              │  │  Admission Webhook     │ │
                              │  │  (pod annotation)      │ │
                              │  └───────────────────────┘ │
                              │  ┌───────────────────────┐ │
                              │  │  DNSPolicy CRD         │ │
                              │  │  (custom resources)    │ │
                              │  └───────────────────────┘ │
                              └────────────────────────────┘
```

---

## Publishing to GitHub Pages

This repository uses [chart-releaser](https://github.com/helm/chart-releaser) via the `helm/chart-releaser-action` GitHub Action. See [`_docs/`](_docs/) for the Jekyll site template used by GitHub Pages.

To release manually:

```bash
# Package individual charts
helm package dashdns           -d .deploy/
helm package dns-mesh-controller -d .deploy/
helm package dns-mesh-full     -d .deploy/

# Generate / update the index
helm repo index .deploy/ --url https://dashdns.github.io/helm-charts

# Push .deploy/ contents to the gh-pages branch
```

---

## Development

### Lint all charts

```bash
helm lint dashdns/
helm lint dns-mesh-controller/
helm lint dns-mesh-full/
```

### Render templates locally

```bash
helm template dns-mesh ./dns-mesh-full/ \
  --namespace dns-mesh \
  --debug
```

### Run with a local dependency

```bash
# From repo root — use local chart paths instead of the remote repo
helm dependency build dns-mesh-full/
helm install dns-mesh dns-mesh-full/ --namespace dns-mesh --create-namespace
```

---

## License

[Apache 2.0](LICENSE)
