# Architecture Overview

## Design Goals

1. Single public endpoint for OSDU APIs only
2. All admin/internal services accessible via VPN only
3. OSDU microservices managed through Istio service mesh
4. Clean separation between public and private network access
5. Secure, production-like setup for learning and experimentation

## High-Level Architecture

```
                              INTERNET
                                  │
               ┌──────────────────┴──────────────────┐
               │                                     │
               ▼                                     ▼
     ┌───────────────────┐                ┌───────────────────┐
     │   Public Access   │                │    VPN Access     │
     │   Ports 80, 443   │                │  Port 51820/UDP   │
     │                   │                │    (WireGuard)    │
     └─────────┬─────────┘                └─────────┬─────────┘
               │                                    │
               ▼                                    ▼
     ┌───────────────────┐                ┌───────────────────┐
     │  osdu.domain.com  │                │  Private Network  │
     │    (public IP)    │                │   10.10.0.0/24    │
     └─────────┬─────────┘                └─────────┬─────────┘
               │                                    │
═══════════════╪════════════════════════════════════╪═══════════════
               │           OSDU SERVER              │
               │        135.181.137.138             │
               │                                    │
┌──────────────┴────────────────────────────────────┴──────────────┐
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Nginx (systemd, host)                   │  │
│  │                                                            │  │
│  │  Public Listener (135.181.137.138:443):                    │  │
│  │    └── osdu.domain.com → Istio Ingress :30080              │  │
│  │                                                            │  │
│  │  Private Listener (10.10.0.1:443):                         │  │
│  │    ├── minio.domain.com   → MinIO :30900                   │  │
│  │    ├── console.domain.com → MinIO Console :30901           │  │
│  │    ├── kibana.domain.com  → Kibana :30561                  │  │
│  │    ├── grafana.domain.com → Grafana :30300                 │  │
│  │    ├── argocd.domain.com  → ArgoCD :30443                  │  │
│  │    ├── kiali.domain.com   → Kiali :30686                   │  │
│  │    └── jaeger.domain.com  → Jaeger :30686                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┴────────────────────┐              │
│         │                                         │              │
│         ▼                                         ▼              │
│  ┌─────────────┐                    ┌─────────────────────────┐  │
│  │   Istio     │                    │   Internal Services     │  │
│  │   Ingress   │                    │   (VPN access only)     │  │
│  │   Gateway   │                    │                         │  │
│  │   :30080    │                    │  ┌───────┐ ┌─────────┐  │  │
│  └──────┬──────┘                    │  │ MinIO │ │ Kibana  │  │  │
│         │                           │  └───────┘ └─────────┘  │  │
│         ▼                           │  ┌───────┐ ┌─────────┐  │  │
│  ┌─────────────────────────┐        │  │Grafana│ │ ArgoCD  │  │  │
│  │    Istio Service Mesh   │        │  └───────┘ └─────────┘  │  │
│  │  ┌───────┐ ┌───────┐    │        │  ┌───────┐ ┌─────────┐  │  │
│  │  │Storage│ │Search │    │        │  │ Kiali │ │ Jaeger  │  │  │
│  │  └───────┘ └───────┘    │        │  └───────┘ └─────────┘  │  │
│  │  ┌───────┐ ┌───────┐    │        └─────────────────────────┘  │
│  │  │ Legal │ │Indexer│ ...│                                     │
│  │  └───────┘ └───────┘    │        ┌─────────────────────────┐  │
│  └─────────────────────────┘        │    Data Layer           │  │
│                                     │  ┌────────┐ ┌────────┐  │  │
│                                     │  │Postgres│ │ Elastic│  │  │
│                                     │  └────────┘ └────────┘  │  │
│                                     │  ┌────────┐ ┌────────┐  │  │
│                                     │  │ Redis  │ │ Kafka  │  │  │
│                                     │  └────────┘ └────────┘  │  │
│                                     └─────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Network Zones

### Public Zone (Internet-Facing)

| Endpoint | Purpose | Authentication |
|----------|---------|----------------|
| `osdu.domain.com` | OSDU Platform APIs | OSDU Auth (OAuth2/Keycloak) |

**Exposed Ports:**
- 80/tcp (HTTP → HTTPS redirect)
- 443/tcp (HTTPS)
- 51820/udp (WireGuard VPN)

### Private Zone (VPN-Only)

| Endpoint | Purpose | Authentication |
|----------|---------|----------------|
| `minio.domain.com` | MinIO S3 API | S3 credentials |
| `console.domain.com` | MinIO Web Console | MinIO login |
| `kibana.domain.com` | Log analysis | Basic auth / built-in |
| `grafana.domain.com` | Metrics dashboards | Grafana login |
| `argocd.domain.com` | GitOps CD | ArgoCD login |
| `kiali.domain.com` | Istio visualization | Basic auth |
| `jaeger.domain.com` | Distributed tracing | Basic auth |

**Access:** VPN connection required (WireGuard)

## Routing Patterns

### Two-Layer Ingress with Network Isolation

```
┌─────────────────────────────────────────────────────────────────┐
│                        Access Patterns                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  External API Client          Admin (You)                       │
│        │                           │                            │
│        │ HTTPS                     │ WireGuard VPN              │
│        │                           │                            │
│        ▼                           ▼                            │
│  ┌───────────┐              ┌────────────┐                      │
│  │ Public IP │              │ VPN Tunnel │                      │
│  │  :443     │              │ 10.10.0.x  │                      │
│  └─────┬─────┘              └──────┬─────┘                      │
│        │                           │                            │
│        ▼                           ▼                            │
│  ┌───────────┐              ┌────────────┐                      │
│  │   Nginx   │              │   Nginx    │                      │
│  │  (public) │              │  (private) │                      │
│  └─────┬─────┘              └──────┬─────┘                      │
│        │                           │                            │
│        ▼                           ▼                            │
│  ┌───────────┐              ┌────────────────────────────────┐  │
│  │   OSDU    │              │  MinIO │ Kibana │ Grafana │ ...│  │
│  │   APIs    │              └────────────────────────────────┘  │
│  └───────────┘                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## DNS Configuration

### Public DNS (Cloudflare)

```
osdu.domain.com    A    135.181.137.138    (proxy OFF)
```

### Private DNS (VPN clients)

Option A: Local DNS server (dnsmasq/Pi-hole on server)
```
minio.domain.com     →  10.10.0.1
console.domain.com   →  10.10.0.1
kibana.domain.com    →  10.10.0.1
grafana.domain.com   →  10.10.0.1
argocd.domain.com    →  10.10.0.1
kiali.domain.com     →  10.10.0.1
jaeger.domain.com    →  10.10.0.1
```

Option B: Client hosts file
```
# /etc/hosts on VPN client
10.10.0.1  minio.domain.com console.domain.com kibana.domain.com
10.10.0.1  grafana.domain.com argocd.domain.com kiali.domain.com
10.10.0.1  jaeger.domain.com
```

## Security Model

### Defense in Depth

```
Layer 1: Network
├── Firewall (UFW) blocks all except 80, 443, 51820
├── Admin services only accessible via VPN
└── VPN uses WireGuard with key-based auth

Layer 2: Transport
├── TLS 1.2/1.3 for all HTTPS
├── WireGuard encryption for VPN traffic
└── mTLS within Istio mesh

Layer 3: Application
├── OSDU OAuth2/OIDC authentication
├── Service-specific auth (Grafana, ArgoCD, etc.)
└── Basic auth for simpler UIs (Kibana, Kiali)
```

### Access Matrix

| Role | Public (osdu.*) | VPN Required | Services |
|------|-----------------|--------------|----------|
| API Client | ✅ | ❌ | OSDU APIs only |
| Developer | ✅ | ✅ | All services |
| Admin | ✅ | ✅ | All services |
| Monitoring | ❌ | ✅ | Grafana, Kibana, Jaeger |

## Next Steps

- [ ] VPN setup (WireGuard configuration)
- [ ] Nginx split listener configuration
- [ ] Firewall rules
- [ ] Private DNS setup
- [ ] Authentication for admin UIs
- [ ] Observability stack design
- [ ] Resource allocation planning
- [ ] CI/CD pipeline design
