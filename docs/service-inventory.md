# Service Inventory

## Overview

This document catalogs all services in the OSDU deployment, their exposure strategy, and resource requirements.

## Service Exposure Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Exposure Zones                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PUBLIC (Internet via 135.181.137.138)                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  osdu.domain.com → OSDU APIs (via Istio)                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  PRIVATE (VPN via 10.10.0.1)                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Admin & Monitoring UIs                                     │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │    │
│  │  │  MinIO  │ │ Kibana  │ │ Grafana │ │ ArgoCD  │           │    │
│  │  │ Console │ │         │ │         │ │         │           │    │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │    │
│  │  ┌─────────┐ ┌─────────┐                                   │    │
│  │  │  Kiali  │ │ Jaeger  │                                   │    │
│  │  └─────────┘ └─────────┘                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  INTERNAL (Cluster only - no external access)                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Data Layer                                                 │    │
│  │  ┌──────────┐ ┌─────────────┐ ┌───────┐ ┌───────┐          │    │
│  │  │PostgreSQL│ │Elasticsearch│ │ Redis │ │ Kafka │          │    │
│  │  └──────────┘ └─────────────┘ └───────┘ └───────┘          │    │
│  │                                                             │    │
│  │  Platform Services                                          │    │
│  │  ┌────────┐ ┌────────────┐ ┌───────────┐                   │    │
│  │  │ Istio  │ │ Prometheus │ │Zookeeper  │                   │    │
│  │  └────────┘ └────────────┘ └───────────┘                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

## Public Services

Services accessible from the internet (OSDU API only).

| Subdomain | Service | Auth | Notes |
|-----------|---------|------|-------|
| `osdu.domain.com` | Istio Ingress | OSDU OAuth2/OIDC | Only public endpoint |

### OSDU Core Services (via Istio)

| Service | Purpose | Endpoint Path | Memory | CPU |
|---------|---------|---------------|--------|-----|
| Storage | Record storage CRUD | `/api/storage/v2/*` | 512Mi-1Gi | 0.5 |
| Search | Query records | `/api/search/v2/*` | 512Mi-1Gi | 0.5 |
| Legal | Legal tag management | `/api/legal/v1/*` | 256Mi-512Mi | 0.25 |
| Indexer | Elasticsearch indexing | `/api/indexer/v2/*` | 512Mi-1Gi | 0.5 |
| Schema | Schema management | `/api/schema-service/v1/*` | 256Mi-512Mi | 0.25 |
| Partition | Data partition mgmt | `/api/partition/v1/*` | 256Mi-512Mi | 0.25 |
| Entitlements | Access control | `/api/entitlements/v2/*` | 256Mi-512Mi | 0.25 |
| File | File handling | `/api/file/v2/*` | 512Mi-1Gi | 0.5 |
| Workflow | Airflow/DAG mgmt | `/api/workflow/v1/*` | 512Mi-1Gi | 0.5 |
| Notification | Event notifications | `/api/notification/v1/*` | 256Mi-512Mi | 0.25 |
| Register | Service registration | `/api/register/v1/*` | 256Mi-512Mi | 0.25 |
| CRS Catalog | Coordinate systems | `/api/crs/catalog/*` | 256Mi-512Mi | 0.25 |
| CRS Conversion | Coord conversion | `/api/crs/converter/*` | 256Mi-512Mi | 0.25 |
| Unit | Unit conversion | `/api/unit/v3/*` | 256Mi-512Mi | 0.25 |

**Total OSDU Services Estimate:** ~6-10GB RAM, 4-6 CPU cores

## Private Services (VPN-Only)

Admin UIs and monitoring tools accessible only via WireGuard VPN.

| Subdomain | Service | NodePort | Auth | Purpose |
|-----------|---------|----------|------|---------|
| `minio.domain.com` | MinIO API | 30900 | S3 credentials | Object Storage API |
| `console.domain.com` | MinIO Console | 30901 | MinIO login | Object Storage UI |
| `kibana.domain.com` | Kibana | 30561 | Built-in | Log analysis |
| `grafana.domain.com` | Grafana | 30300 | Built-in | Metrics dashboards |
| `argocd.domain.com` | ArgoCD | 30443 | Built-in | GitOps CD |
| `kiali.domain.com` | Kiali | 30686 | Token/Basic | Istio visualization |
| `jaeger.domain.com` | Jaeger | 30686 | None/Basic | Distributed tracing |

### Resource Estimates (Private Services)

| Service | Memory | CPU | Storage |
|---------|--------|-----|---------|
| MinIO | 1-2Gi | 1 | 200Gi+ |
| Kibana | 512Mi-1Gi | 0.5 | - |
| Grafana | 256Mi-512Mi | 0.25 | - |
| ArgoCD | 512Mi-1Gi | 0.5 | - |
| Kiali | 256Mi-512Mi | 0.25 | - |
| Jaeger | 512Mi-1Gi | 0.5 | - |

## Internal Services (Cluster Only)

Services with no external access, only reachable within Kubernetes.

### Data Layer

| Service | Purpose | Memory | CPU | Storage |
|---------|---------|--------|-----|---------|
| PostgreSQL | OSDU metadata | 2-4Gi | 1 | 50Gi |
| Elasticsearch | Search index + logs | 4-8Gi | 2 | 100Gi |
| Redis | Caching | 1-2Gi | 0.5 | - |
| Kafka | Event streaming | 2-4Gi | 1 | 20Gi |
| Zookeeper | Kafka coordination | 512Mi | 0.5 | 5Gi |

### Platform Services

| Service | Purpose | Memory | CPU |
|---------|---------|--------|-----|
| Istio (Pilot) | Service mesh control | 1-2Gi | 1 |
| Prometheus | Metrics collection | 2-4Gi | 1 |
| Cert-Manager | Certificate mgmt | 256Mi | 0.25 |

## Access Matrix

| Role | Public (osdu.*) | VPN (admin UIs) | Cluster Internal |
|------|:---------------:|:---------------:|:----------------:|
| External API Client | ✅ | ❌ | ❌ |
| Developer (VPN) | ✅ | ✅ | ❌ |
| Admin (VPN) | ✅ | ✅ | ❌ |
| Kubernetes Pods | ✅ | ✅ | ✅ |

## Port Allocation Summary

### Firewall Ports (External)

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH (consider restricting to VPN) |
| 80 | TCP | HTTP → HTTPS redirect |
| 443 | TCP | HTTPS (OSDU API) |
| 51820 | UDP | WireGuard VPN |

### NodePort Allocation (Internal)

| Range | Service Type |
|-------|--------------|
| 30080-30099 | Istio Ingress |
| 30300-30399 | Monitoring (Grafana, Prometheus) |
| 30400-30499 | GitOps (ArgoCD) |
| 30500-30599 | Logging (Kibana) |
| 30600-30699 | Tracing (Jaeger, Kiali) |
| 30900-30999 | Storage (MinIO) |

## Resource Summary

### Total Estimated Requirements

| Category | Memory | CPU | Storage |
|----------|--------|-----|---------|
| OSDU Services | 8-12Gi | 4-6 | - |
| Data Layer | 10-18Gi | 5-7 | 375Gi |
| Observability | 4-8Gi | 2-3 | 50Gi |
| Platform | 2-4Gi | 2-3 | - |
| Admin UIs | 2-4Gi | 2-3 | - |
| **Total** | **26-46Gi** | **15-22** | **425Gi** |

### Server Capacity Check

| Resource | Available | Estimated Use | Headroom |
|----------|-----------|---------------|----------|
| Memory | 64GB | 30-46GB | 18-34GB ✅ |
| CPU | 12 threads | 15-22 cores | ⚠️ Overcommit |
| Storage | 700GB (/data) | 425GB | 275GB ✅ |

**Note:** CPU overcommit is acceptable for dev/learning. Production would need dedicated resources.

## Namespace Organization

```
Kubernetes Namespaces:
├── osdu-core/          # OSDU microservices
├── osdu-data/          # PostgreSQL, Elasticsearch, Redis, Kafka
├── osdu-storage/       # MinIO
├── istio-system/       # Istio control plane
├── monitoring/         # Prometheus, Grafana
├── logging/            # Elasticsearch (logs), Kibana
├── tracing/            # Jaeger
├── argocd/             # ArgoCD
└── cert-manager/       # Cert-Manager
```

## Network Policies (Future)

Consider implementing Kubernetes Network Policies to further restrict pod-to-pod communication:

```yaml
# Example: Only allow OSDU services to reach Elasticsearch
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-access
  namespace: osdu-data
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: osdu-core
```
