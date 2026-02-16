# OSDU System Design

Architecture documentation for a self-hosted OSDU Platform deployment on a single Hetzner server.

## Server Specs

| Component | Specification |
|-----------|---------------|
| **Host** | Hetzner Auction Server |
| **CPU** | Xeon E-2176G (6c/12t) |
| **RAM** | 64GB ECC |
| **Storage** | 2x960GB NVMe RAID1 |
| **OS** | Ubuntu 22.04.5 LTS |
| **IP** | 135.181.137.138 |

## Architecture Highlights

- **Public endpoint**: OSDU API only (`osdu.domain.com`)
- **Private access**: Admin UIs via WireGuard VPN
- **Edge proxy**: Nginx with split public/private listeners
- **Service mesh**: Istio for OSDU microservices
- **Observability**: Prometheus, Grafana, ELK, Jaeger

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture-overview.md) | High-level system architecture |
| [Network Security](docs/network-security.md) | VPN, firewall, and network isolation |
| [Ingress Design](docs/ingress-design.md) | Nginx configuration (public/private split) |
| [Service Inventory](docs/service-inventory.md) | Services and exposure strategy |
| [Decisions](docs/decisions.md) | Architecture Decision Records (ADRs) |

## Quick Architecture

```
                    INTERNET
                        │
         ┌──────────────┴──────────────┐
         │                             │
         ▼                             ▼
   ┌───────────┐                ┌────────────┐
   │  Public   │                │    VPN     │
   │  :443     │                │ WireGuard  │
   └─────┬─────┘                └─────┬──────┘
         │                            │
         ▼                            ▼
   ┌───────────┐              ┌─────────────────┐
   │ osdu.*    │              │ minio.*         │
   │ (Nginx)   │              │ kibana.*        │
   └─────┬─────┘              │ grafana.*       │
         │                    │ argocd.*        │
         ▼                    │ (Nginx on VPN)  │
   ┌───────────┐              └────────┬────────┘
   │  Istio    │                       │
   │  Ingress  │              ┌────────┴────────┐
   └─────┬─────┘              │ Admin Services  │
         │                    │ MinIO, Kibana,  │
         ▼                    │ Grafana, etc.   │
   ┌───────────┐              └─────────────────┘
   │   OSDU    │
   │ Services  │
   └───────────┘
```

## Key Design Decisions

1. **ADR-001**: Edge Proxy Pattern (Nginx + Istio)
2. **ADR-002**: Wildcard SSL with Let's Encrypt
3. **ADR-003**: Nginx on host (not in K8s)
4. **ADR-004**: NodePort for service exposure
5. **ADR-005**: Cloudflare DNS
6. **ADR-006**: WireGuard VPN for private access
7. **ADR-007**: Explicit IP binding in Nginx

## Quick Links

- [OSDU Official Docs](https://community.opengroup.org/osdu)
- [Istio Documentation](https://istio.io/latest/docs/)
- [WireGuard Documentation](https://www.wireguard.com/)

## Status

🚧 **In Development** — Architecture design phase

### Completed
- [x] High-level architecture
- [x] Ingress pattern (Nginx + Istio)
- [x] Network isolation (VPN design)
- [x] Service inventory
- [x] ADRs (7 accepted)

### Pending
- [ ] Domain purchase and DNS setup
- [ ] WireGuard implementation
- [ ] Nginx configuration deployment
- [ ] OSDU authentication design
- [ ] Backup strategy
- [ ] CI/CD pipeline
