# Service Inventory

## Overview

This document catalogs all services in the OSDU deployment, their exposure strategy, and resource requirements.

## Service Categories

```
┌─────────────────────────────────────────────────────────────────┐
│                        Service Layers                           │
├─────────────────────────────────────────────────────────────────┤
│  External Access (via Nginx subdomains)                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │  OSDU   │ │  MinIO  │ │ Kibana  │ │ Grafana │ │ ArgoCD  │   │
│  │   API   │ │ Console │ │         │ │         │ │         │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  OSDU Core Services (via Istio mesh)                            │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │Storage │ │ Search │ │ Legal  │ │Indexer │ │  CRS   │  ...   │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘        │
├─────────────────────────────────────────────────────────────────┤
│  Data Layer (internal only)                                     │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │ PostgreSQL │ │Elasticsearch│ │   Redis    │ │   MinIO    │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  Platform Services (internal only)                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                  │
│  │   Istio    │ │ Prometheus │ │   Kafka    │                  │
│  └────────────┘ └────────────┘ └────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

## OSDU Core Services

Services managed by Istio, accessed via OSDU API gateway.

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

## Data Layer Services

Backend services, internal only (ClusterIP).

| Service | Purpose | Exposure | Memory | CPU | Storage |
|---------|---------|----------|--------|-----|---------|
| PostgreSQL | OSDU metadata | Internal | 2-4Gi | 1 | 50Gi |
| Elasticsearch | Search index | Internal | 4-8Gi | 2 | 100Gi |
| Redis | Caching | Internal | 1-2Gi | 0.5 | - |
| MinIO | Object storage | Internal + Console | 1-2Gi | 1 | 200Gi+ |
| Kafka | Event streaming | Internal | 2-4Gi | 1 | 20Gi |
| Zookeeper | Kafka coordination | Internal | 512Mi | 0.5 | 5Gi |

**Total Data Layer Estimate:** ~12-20GB RAM, 6-8 CPU cores

## Observability Stack

| Service | Purpose | Exposure | Memory | CPU |
|---------|---------|----------|--------|-----|
| Prometheus | Metrics collection | Internal | 2-4Gi | 1 |
| Grafana | Metrics visualization | Subdomain | 256Mi-512Mi | 0.25 |
| Elasticsearch | Log storage | Internal | (shared above) | - |
| Kibana | Log visualization | Subdomain | 512Mi-1Gi | 0.5 |
| Jaeger | Distributed tracing | Internal/Subdomain | 512Mi-1Gi | 0.5 |
| Kiali | Istio visualization | Internal/Subdomain | 256Mi-512Mi | 0.25 |

## Platform Services

| Service | Purpose | Exposure | Memory | CPU |
|---------|---------|----------|--------|-----|
| Istio (Pilot) | Service mesh control | Internal | 1-2Gi | 1 |
| Istio (Ingress) | Mesh ingress | NodePort | 256Mi-512Mi | 0.5 |
| ArgoCD | GitOps CD | Subdomain | 512Mi-1Gi | 0.5 |
| Cert-Manager | Certificate mgmt | Internal | 256Mi | 0.25 |

## Exposure Strategy

### External (Subdomain)

Services accessible from the internet via Nginx reverse proxy.

| Subdomain | Service | Auth Required | Notes |
|-----------|---------|---------------|-------|
| `osdu.*` | Istio Ingress | Yes (OSDU auth) | Main API endpoint |
| `minio.*` | MinIO API | Yes (S3 auth) | S3-compatible API |
| `console.*` | MinIO Console | Yes | Web UI for MinIO |
| `kibana.*` | Kibana | Yes (basic auth) | Log analysis |
| `grafana.*` | Grafana | Yes (built-in) | Metrics dashboards |
| `argocd.*` | ArgoCD | Yes (built-in) | GitOps UI |
| `kiali.*` | Kiali | Yes (basic auth) | Istio visualization |
| `jaeger.*` | Jaeger | Yes (basic auth) | Tracing UI |

### Internal Only (ClusterIP)

Services only accessible within the Kubernetes cluster.

- PostgreSQL
- Elasticsearch (API)
- Redis
- Kafka
- Zookeeper
- Prometheus
- Istio control plane

## Resource Summary

### Total Estimated Requirements

| Category | Memory | CPU | Storage |
|----------|--------|-----|---------|
| OSDU Services | 8-12Gi | 4-6 | - |
| Data Layer | 12-20Gi | 6-8 | 400Gi |
| Observability | 4-8Gi | 2-3 | 50Gi |
| Platform | 2-4Gi | 2-3 | - |
| **Total** | **26-44Gi** | **14-20** | **450Gi** |

### Server Capacity

| Resource | Available | Estimated Use | Headroom |
|----------|-----------|---------------|----------|
| Memory | 64GB | 30-44GB | 20-34GB |
| CPU | 12 threads | 14-20 cores | ⚠️ Overcommit OK |
| Storage | 700GB (/data) | 450GB | 250GB |

**Note:** CPU overcommit is acceptable for dev/learning environments. In production, you'd want dedicated cores.

## Namespace Organization

```
osdu-system/
├── osdu-core/          # OSDU microservices
├── osdu-data/          # PostgreSQL, Elasticsearch, Redis, Kafka
├── osdu-storage/       # MinIO
├── istio-system/       # Istio control plane
├── monitoring/         # Prometheus, Grafana
├── logging/            # Elasticsearch (logs), Kibana
├── argocd/             # ArgoCD
└── cert-manager/       # Cert-Manager
```
