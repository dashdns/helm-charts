# dns-mesh-controller

Kubernetes controller that manages `DNSPolicy` CRDs and exposes a policy API consumed by DashDNS. Also includes an **admission webhook** to annotate pods at creation time.

[![Helm](https://img.shields.io/badge/helm-3.x-blue)](https://helm.sh)
[![License](https://img.shields.io/github/license/dashdns/helm-charts)](../LICENSE)

## Add the Helm Repository

```bash
helm repo add dashdns https://dashdns.github.io/helm-charts
helm repo update
```

## Quick install

```bash
helm install dns-mesh-controller dashdns/dns-mesh-controller \
  --namespace dns-mesh-controller \
  --create-namespace
```

## Key values

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

## Webhook TLS certificates

The admission webhook requires a valid TLS certificate. Pass the base64-encoded values:

```bash
helm install dns-mesh-controller dashdns/dns-mesh-controller \
  --set webhook.certChain.ca="$(cat ca.crt | base64)" \
  --set webhook.certChain.cert="$(cat tls.crt | base64)" \
  --set webhook.certChain.key="$(cat tls.key | base64)"
```

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

## Development

### Lint

```bash
helm lint dns-mesh-controller/
```

### Render templates locally

```bash
helm template dns-mesh-controller ./dns-mesh-controller/ --namespace dns-mesh --debug
```

---

[Apache 2.0](../LICENSE)
