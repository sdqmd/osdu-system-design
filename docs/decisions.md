# Architecture Decision Records (ADRs)

## Overview

This document captures key architectural decisions made for the OSDU deployment.

---

## ADR-001: Edge Proxy Pattern

**Status:** Accepted  
**Date:** 2026-02-17

### Context

We need to expose multiple services (OSDU APIs, MinIO, Kibana, Grafana) through a single server with different subdomains. OSDU uses Istio for its service mesh.

### Options Considered

1. **Path-based routing through Istio**
   - All traffic goes through Istio Ingress
   - Different paths route to different services
   - ❌ Some apps don't work well with path prefixes
   - ❌ Requires all services in the mesh

2. **Subdomain routing through Istio**
   - Multiple Istio Gateways with host-based routing
   - ❌ Adds mesh complexity for non-OSDU services
   - ❌ MinIO/Kibana don't need mesh features

3. **Edge Proxy (Nginx) + Istio** ✅
   - Nginx handles subdomain routing and SSL
   - Routes OSDU traffic to Istio
   - Routes supporting services directly
   - ✅ Clean separation of concerns
   - ✅ Standard production pattern

### Decision

Use Nginx as an edge proxy on the host, routing to Istio for OSDU services and directly to other services (MinIO, Kibana, etc.) via NodePorts.

### Consequences

- **Positive:** Simple, battle-tested, easy to debug
- **Positive:** Supporting services don't need mesh injection
- **Positive:** Single point for SSL certificate management
- **Negative:** Another component to manage (Nginx outside K8s)

---

## ADR-002: SSL/TLS Strategy

**Status:** Accepted  
**Date:** 2026-02-17

### Context

Need to secure all services with HTTPS. Multiple subdomains will be used.

### Options Considered

1. **Individual certificates per subdomain**
   - Let's Encrypt HTTP challenge for each
   - ❌ Tedious to manage many certs
   - ❌ Each new subdomain needs new cert

2. **Wildcard certificate** ✅
   - Let's Encrypt with DNS challenge
   - ✅ Single cert for all subdomains
   - ✅ Easy to add new subdomains
   - Requires DNS provider API access

3. **Cloudflare proxy (origin cert)**
   - Cloudflare handles public SSL
   - Origin cert for Cloudflare ↔ Server
   - ❌ Adds dependency on Cloudflare proxy

### Decision

Use Let's Encrypt wildcard certificate (`*.domain.com`) with Cloudflare DNS challenge.

### Consequences

- **Positive:** Single cert covers all subdomains
- **Positive:** Easy to add new services
- **Negative:** Requires Cloudflare API token with DNS edit permissions

---

## ADR-003: Load Balancer Placement

**Status:** Accepted  
**Date:** 2026-02-17

### Context

Single-server deployment needs to handle incoming traffic routing. Options range from external cloud LBs to host-based proxies.

### Options Considered

1. **Hetzner Load Balancer (external)**
   - Separate LB service (~€5/mo)
   - ❌ Overkill for single server
   - ❌ Additional cost

2. **Nginx inside Kubernetes**
   - Nginx Ingress Controller with hostNetwork
   - ❌ Conflicts/overlaps with Istio
   - ❌ More complex networking

3. **Nginx on host (systemd)** ✅
   - Simple reverse proxy outside K8s
   - ✅ Easy to configure and debug
   - ✅ Clear separation from mesh

### Decision

Run Nginx on the host as a systemd service, listening on ports 80/443.

### Consequences

- **Positive:** Simple operations, easy to troubleshoot
- **Positive:** No conflict with Istio
- **Negative:** Not managed by Kubernetes (manual updates)

---

## ADR-004: Service Exposure Method

**Status:** Accepted  
**Date:** 2026-02-17

### Context

Services in Kubernetes need to be accessible from the host Nginx proxy.

### Options Considered

1. **LoadBalancer service type**
   - ❌ No cloud provider = no external LB provisioning
   - Would need MetalLB for bare metal

2. **NodePort services** ✅
   - Expose on high ports (30000-32767)
   - Accessible from host via localhost
   - ✅ Simple, works on any K8s

3. **Host networking**
   - Pods bind directly to host ports
   - ❌ Port conflicts, less isolation

### Decision

Use NodePort services for all externally-exposed services. Nginx proxies to `127.0.0.1:<nodeport>`.

### Consequences

- **Positive:** Works on any Kubernetes (no cloud dependencies)
- **Positive:** Services remain isolated in cluster network
- **Negative:** Limited port range (30000-32767)

---

## ADR-005: DNS Provider

**Status:** Accepted  
**Date:** 2026-02-17

### Context

Need a DNS provider for the domain that supports:
- API access for Let's Encrypt DNS challenge
- Fast propagation
- Reliability

### Options Considered

1. **Cloudflare (free tier)** ✅
   - ✅ Free, fast propagation
   - ✅ Excellent API for certbot
   - ✅ Optional proxy/DDoS protection
   - Widely used, well-documented

2. **Hetzner DNS**
   - Free with Hetzner account
   - Less mature API tooling
   - ❌ Fewer certbot plugins

3. **Route53 (AWS)**
   - ❌ Costs money
   - ❌ Overkill for this use case

### Decision

Use Cloudflare DNS (free tier) with proxy disabled (DNS only mode).

### Consequences

- **Positive:** Free, fast, reliable
- **Positive:** Easy wildcard cert setup
- **Positive:** Can enable proxy later if needed for DDoS protection

---

## ADR-006: Network Isolation with VPN

**Status:** Accepted  
**Date:** 2026-02-17

### Context

Admin interfaces (Kibana, Grafana, MinIO Console, ArgoCD) should not be exposed to the public internet. Only the OSDU API should be publicly accessible.

### Options Considered

1. **Basic auth on public endpoints**
   - Simple password protection
   - ❌ Still exposed to internet attacks
   - ❌ Brute force risk, credential stuffing

2. **OAuth2 Proxy**
   - SSO for all services
   - ❌ Complex setup
   - ❌ Still publicly accessible login page

3. **IP allowlisting**
   - Whitelist specific IPs
   - ❌ Dynamic IPs cause access issues
   - ❌ Pain when traveling

4. **WireGuard VPN** ✅
   - Services only accessible via VPN
   - ✅ Strong encryption
   - ✅ Fast, kernel-level performance
   - ✅ Works from anywhere
   - ✅ Mobile support (iOS/Android)

5. **Tailscale**
   - Managed WireGuard
   - ✅ Easier setup
   - ❌ External dependency
   - ❌ Free tier device limits

### Decision

Use WireGuard VPN for private network access. Public services (OSDU API) bind to public IP. Private services (admin UIs) bind to VPN IP only.

### Network Design

```
Public (135.181.137.138):
  └── osdu.domain.com (OSDU APIs)

Private (10.10.0.1 via WireGuard):
  ├── minio.domain.com
  ├── console.domain.com
  ├── kibana.domain.com
  ├── grafana.domain.com
  ├── argocd.domain.com
  ├── kiali.domain.com
  └── jaeger.domain.com
```

### Consequences

- **Positive:** Zero attack surface for admin interfaces
- **Positive:** No credential stuffing or brute force possible
- **Positive:** Works from any location with VPN connection
- **Positive:** Mobile access with WireGuard apps
- **Negative:** Requires VPN client setup on each device
- **Negative:** Additional key management for VPN peers

---

## ADR-007: Nginx IP Binding Strategy

**Status:** Accepted  
**Date:** 2026-02-17

### Context

With separate public and private networks, Nginx needs to listen on different interfaces for different services.

### Options Considered

1. **Listen on all interfaces (0.0.0.0)**
   - `listen 443;`
   - ❌ Private services accessible on public IP
   - ❌ Relies solely on firewall for protection

2. **Explicit IP binding** ✅
   - `listen 135.181.137.138:443;` for public
   - `listen 10.10.0.1:443;` for private
   - ✅ Defense in depth
   - ✅ Services unreachable even if firewall misconfigured

### Decision

Bind Nginx listeners to explicit IP addresses:
- Public services: bind to `135.181.137.138:443`
- Private services: bind to `10.10.0.1:443` (VPN interface)

### Consequences

- **Positive:** Strong network isolation at application layer
- **Positive:** Defense in depth (complements firewall rules)
- **Negative:** Must remember to use explicit IPs in config

---

## ADR-008: Private DNS Resolution

**Status:** Proposed  
**Date:** 2026-02-17

### Context

VPN clients need to resolve private service hostnames (e.g., `kibana.domain.com`) to the VPN server IP (`10.10.0.1`).

### Options Considered

1. **Client hosts file**
   - Manual entries on each client
   - ✅ Simple, no server config
   - ❌ Must update each client for changes

2. **dnsmasq on server** ✅
   - Lightweight DNS server
   - VPN clients use server as DNS
   - ✅ Centralized management
   - ✅ Automatic for all clients

3. **Pi-hole on server**
   - DNS + ad blocking
   - ✅ Nice UI
   - ❌ Heavier than needed

### Decision

Use dnsmasq on the server for private DNS resolution. VPN clients configure `DNS = 10.10.0.1` in WireGuard config.

### Consequences

- **Positive:** Add new services without touching clients
- **Positive:** Consistent resolution for all VPN users
- **Negative:** Extra service to maintain

---

## ADR-009: Observability Stack

**Status:** Proposed  
**Date:** 2026-02-17

### Context

Need logging, metrics, and tracing for OSDU platform operations.

### Components

| Concern | Tool | Rationale |
|---------|------|-----------|
| Metrics | Prometheus + Grafana | Standard K8s stack, Istio integration |
| Logging | Elasticsearch + Kibana | Required by OSDU, already in stack |
| Tracing | Jaeger | Istio-native, lighter than Zipkin |
| Mesh Viz | Kiali | Istio service graph |

### Decision

Use the standard Prometheus/Grafana/ELK stack with Jaeger for tracing and Kiali for mesh visualization.

### Consequences

- **Positive:** Well-integrated with Istio
- **Positive:** OSDU uses Elasticsearch anyway
- **Negative:** Memory-heavy (Elasticsearch especially)

---

## Pending Decisions

### ADR-010: Authentication for OSDU APIs
- OAuth2/OIDC provider
- Options: Keycloak, Auth0, Azure AD mock

### ADR-011: Backup Strategy
- PostgreSQL backups
- Elasticsearch snapshots
- MinIO bucket replication

### ADR-012: CI/CD Pipeline
- ArgoCD for GitOps?
- How to update OSDU services?

### ADR-013: Resource Limits and Quotas
- Per-service limits
- Namespace quotas
- Priority classes
