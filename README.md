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

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture-overview.md) | High-level system architecture |
| [Ingress Design](docs/ingress-design.md) | Nginx + Istio ingress pattern |
| [Service Inventory](docs/service-inventory.md) | Services and exposure strategy |
| [Decisions](docs/decisions.md) | Architecture Decision Records (ADRs) |

## Quick Links

- [OSDU Official Docs](https://community.opengroup.org/osdu)
- [Istio Documentation](https://istio.io/latest/docs/)

## Status

🚧 **In Development** — Architecture design phase
