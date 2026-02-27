# dashdns

eBPF-based DNS proxy that attaches XDP/TC programs to a network interface. Runs as a **DaemonSet** so every node intercepts and filters DNS traffic.

[![Helm](https://img.shields.io/badge/helm-3.x-blue)](https://helm.sh)
[![License](https://img.shields.io/github/license/dashdns/helm-charts)](../LICENSE)

## Add the Helm Repository

```bash
helm repo add dashdns https://dashdns.github.io/helm-charts
helm repo update
```

## Quick install

```bash
helm install dashdns dashdns/dashdns \
  --namespace dashdns \
  --create-namespace
```

## Key values

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

## Security context

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

## Enable Prometheus metrics

```bash
helm install dashdns dashdns/dashdns \
  --set prometheus.enabled=true \
  --set prometheus.serviceMonitorDiscoverLabels.release=prometheus-operator
```

## Development

### Lint

```bash
helm lint dashdns/
```

### Render templates locally

```bash
helm template dashdns ./dashdns/ --namespace dashdns --debug
```

---

[Apache 2.0](../LICENSE)
