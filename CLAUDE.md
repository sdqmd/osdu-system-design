# CLAUDE.md - Project Context for AI Agents

## Project Overview

**Repository:** `osdu-system-design`
**Purpose:** Architecture documentation for a self-hosted OSDU Platform deployment on a single Hetzner server.
**Owner:** Sadiq (sdqmd)
**Status:** Design phase — no implementation yet

## What is OSDU?

OSDU (Open Subsurface Data Universe) is an open-source data platform for the energy industry. It provides:
- Standardized APIs for subsurface data
- Microservices architecture
- Runs on Kubernetes with Istio service mesh

## Target Environment

| Component | Specification |
|-----------|---------------|
| Server | Hetzner Auction (single node) |
| IP | 135.181.137.138 |
| CPU | Xeon E-2176G (6c/12t) |
| RAM | 64GB ECC |
| Storage | 2x960GB NVMe RAID1 |
| OS | Ubuntu 22.04.5 LTS |

## Architecture Summary

```
Internet
    │
    ├── Public (135.181.137.138)
    │     ├── osdu.domain.com → OSDU APIs (via Istio)
    │     └── auth.domain.com → Keycloak login (limited endpoints)
    │
    └── VPN (WireGuard, 10.10.0.0/24)
          ├── keycloak.domain.com → Keycloak Admin
          ├── minio.domain.com → MinIO
          ├── kibana.domain.com → Kibana
          ├── grafana.domain.com → Grafana
          └── argocd.domain.com → ArgoCD
```

**Key Pattern:** Nginx edge proxy on host → Istio for OSDU services, NodePort for others. Private services only accessible via WireGuard VPN.

## Documentation Structure

```
docs/
├── architecture-overview.md   # High-level architecture, network zones
├── network-security.md        # WireGuard VPN, firewall, private DNS
├── ingress-design.md          # Nginx configs (public/private split)
├── authentication.md          # Keycloak, JWT validation, entitlements
├── backup-strategy.md         # DRAFT - options documented, decisions pending
├── service-inventory.md       # All services, ports, resources
└── decisions.md               # Architecture Decision Records (ADRs)
```

## Current Progress

### Completed (Decisions Made)
- [x] ADR-001: Edge Proxy Pattern (Nginx + Istio)
- [x] ADR-002: Wildcard SSL with Let's Encrypt + Cloudflare DNS
- [x] ADR-003: Nginx on host (systemd, not in K8s)
- [x] ADR-004: NodePort for service exposure
- [x] ADR-005: Cloudflare DNS (free tier)
- [x] ADR-006: WireGuard VPN for private access
- [x] ADR-007: Explicit IP binding in Nginx
- [x] ADR-008: dnsmasq for private DNS (proposed)
- [x] ADR-009: Observability stack (Prometheus/Grafana/ELK/Jaeger)
- [x] ADR-010: Keycloak as Identity Provider
- [x] ADR-011: Split auth exposure (public login, private admin)

### Pending (Open Questions)
- [ ] ADR-012: Backup Strategy — See `docs/backup-strategy.md` for options
- [ ] ADR-013: CI/CD Pipeline — ArgoCD? Deployment strategy?
- [ ] ADR-014: Resource Limits — Per-service limits, quotas
- [ ] ADR-015: Secret Management — K8s secrets vs Vault

### Not Yet Started
- Domain purchase
- Actual implementation
- Kubernetes cluster setup
- OSDU deployment

## Key Design Decisions to Remember

1. **Only OSDU API is public** — All admin UIs behind VPN
2. **Nginx binds to explicit IPs** — `135.181.137.138:443` for public, `10.10.0.1:443` for private
3. **Keycloak split exposure** — Login endpoints public, admin console VPN-only
4. **WireGuard for VPN** — Not Tailscale, full control preferred
5. **Single wildcard SSL cert** — `*.domain.com` via Cloudflare DNS challenge

## Port Allocation

| Service | NodePort | Notes |
|---------|----------|-------|
| Istio Ingress | 30080 | OSDU APIs |
| Keycloak | 30800 | Auth |
| MinIO API | 30900 | S3 |
| MinIO Console | 30901 | Web UI |
| Grafana | 30300 | Metrics |
| Kibana | 30561 | Logs |
| Kiali/Jaeger | 30686 | Tracing |

## How to Continue This Work

### Adding New Design Documents
1. Create `docs/<topic>.md`
2. Update `README.md` documentation table
3. Add ADR entry in `docs/decisions.md`
4. Commit with descriptive message

### Making Decisions on Open Questions
1. Read relevant draft document (e.g., `backup-strategy.md`)
2. Discuss options with user
3. Update document with decision
4. Move ADR from "Pending" to accepted with rationale
5. Update any affected documents

### Document Style
- Use Markdown tables for comparisons
- Include ASCII diagrams for architecture
- Keep ADRs structured: Context → Options → Decision → Consequences
- Mark drafts clearly with "Status: Draft"

## Commands

```bash
# Clone repo
git clone git@github.com:sdqmd/osdu-system-design.git

# After changes
git add -A
git commit -m "Descriptive message"
git push
```

## Related Resources

- OSDU GitLab: https://community.opengroup.org/osdu
- Istio docs: https://istio.io/latest/docs/
- Keycloak docs: https://www.keycloak.org/documentation
- WireGuard docs: https://www.wireguard.com/

## User Preferences

- Prefers discussion before implementation ("plan first, don't execute")
- Wants open questions documented, not prematurely decided
- Values clear architecture documentation
- Uses GitHub (sdqmd account)

---

*Last updated: 2026-02-17*
