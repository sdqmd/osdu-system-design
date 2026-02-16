# Architecture Overview

## Design Goals

1. Single endpoint with multiple services accessible via subdomains
2. OSDU microservices managed through Istio service mesh
3. Supporting services (MinIO, Kibana, Grafana) accessible outside the mesh
4. Clean separation of concerns between edge routing and service mesh
5. Secure, production-like setup for learning and experimentation

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    OSDU Server (135.181.137.138)                   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                 Nginx (systemd, host)                        │  │
│  │                 Ports: 80, 443                               │  │
│  │                 Cert: *.osdu.yourdomain.com                  │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │                                      │
│         ┌───────────────────┼───────────────────┐                  │
│         │                   │                   │                  │
│         ▼                   ▼                   ▼                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │ osdu.*      │    │ minio.*     │    │ kibana.*    │             │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘             │
│         │                  │                  │                    │
│  ═══════╪══════════════════╪══════════════════╪═══════════════     │
│  │      │    Kubernetes    │                  │              │     │
│  │      ▼                  ▼                  ▼              │     │
│  │ ┌─────────┐       ┌─────────┐       ┌──────────┐          │     │
│  │ │ Istio   │       │ MinIO   │       │ Kibana   │          │     │
│  │ │ Ingress │       │ :9000   │       │ :5601    │          │     │
│  │ │ Gateway │       │ :9001   │       └──────────┘          │     │
│  │ │ :30080  │       │ (console)│                            │     │
│  │ └────┬────┘       └─────────┘                             │     │
│  │      │                                                    │     │
│  │      ▼                                                    │     │
│  │ ┌─────────────────────────────────────────────┐           │     │
│  │ │              Istio Service Mesh             │           │     │
│  │ │  ┌─────────┐ ┌─────────┐ ┌─────────┐        │           │     │
│  │ │  │ Storage │ │ Search  │ │ Legal   │  ...   │           │     │
│  │ │  │ Service │ │ Service │ │ Service │        │           │     │
│  │ │  └─────────┘ └─────────┘ └─────────┘        │           │     │
│  │ └─────────────────────────────────────────────┘           │     │
│  │                                                           │     │
│  │  ┌──────────────────────────────────────────┐             │     │
│  │  │          Supporting Services             │             │     │
│  │  │  ┌────────┐ ┌────────┐ ┌─────────────┐   │             │     │
│  │  │  │ Elastic│ │ Redis  │ │ PostgreSQL  │   │             │     │
│  │  │  └────────┘ └────────┘ └─────────────┘   │             │     │
│  │  └──────────────────────────────────────────┘             │     │
│  ═════════════════════════════════════════════════════════════     │
└────────────────────────────────────────────────────────────────────┘
```

## Routing Patterns

### Pattern: Edge Proxy + Service Mesh

This architecture uses a **two-layer ingress pattern**:

1. **Edge Layer (Nginx)**: Handles north-south traffic
   - SSL/TLS termination
   - Subdomain-based routing
   - Static file serving (if needed)
   - Rate limiting & basic security

2. **Mesh Layer (Istio)**: Handles east-west traffic
   - Service-to-service mTLS
   - Traffic management (retries, circuit breaking)
   - Observability (tracing, metrics)
   - Canary deployments

### Why This Pattern?

| Concern | Handled By |
|---------|------------|
| SSL termination | Nginx (edge) |
| Subdomain routing | Nginx (edge) |
| DDoS protection | Nginx (edge) / Cloudflare |
| Service discovery | Istio (mesh) |
| mTLS between services | Istio (mesh) |
| Traffic splitting | Istio (mesh) |
| Observability | Istio (mesh) |

This is a **standard production pattern** used in cloud environments:
- AWS: ALB/NLB → Istio Ingress
- GCP: Cloud Load Balancer → Istio Ingress
- Azure: Azure LB/App Gateway → Istio Ingress
- On-Prem: Nginx/HAProxy → Istio Ingress

## DNS Configuration

```
DNS Provider: Cloudflare (recommended)

Records:
  *.osdu.yourdomain.com  →  A  →  135.181.137.138  (proxy OFF)
```

Using Cloudflare with proxy OFF (gray cloud) allows:
- Nginx to handle SSL directly
- Certbot DNS challenge for wildcard certs
- Optional: Enable proxy later for DDoS protection

## Subdomain Mapping

| Subdomain | Target | Port | Purpose |
|-----------|--------|------|---------|
| `osdu.domain.com` | Istio Ingress | 30080 | OSDU Platform APIs |
| `minio.domain.com` | MinIO API | 30900 | Object Storage API |
| `console.domain.com` | MinIO Console | 30901 | MinIO Web UI |
| `kibana.domain.com` | Kibana | 30561 | Log Analysis |
| `grafana.domain.com` | Grafana | 30300 | Metrics Dashboard |
| `argocd.domain.com` | ArgoCD | 30443 | GitOps CD |

## Next Steps

- [ ] Service inventory and exposure strategy
- [ ] Authentication design (protecting admin UIs)
- [ ] Observability stack design
- [ ] Resource allocation planning
- [ ] CI/CD pipeline design
