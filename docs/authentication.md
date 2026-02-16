# Authentication Design

## Overview

This document details the authentication and authorization architecture for the OSDU platform using Keycloak as the identity provider.

## Authentication Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Authentication Flow                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐     1. Login      ┌─────────────────┐                    │
│  │  User  │ ─────────────────▶│    Keycloak     │                    │
│  │        │◀───────────────── │ (Identity Prov) │                    │
│  └───┬────┘   2. JWT Token    └─────────────────┘                    │
│      │                               ▲                               │
│      │ 3. API Request                │ 5. Validate Token             │
│      │    + Bearer Token             │    (JWKS)                     │
│      ▼                               │                               │
│  ┌─────────────────┐          ┌──────┴────────┐                      │
│  │  OSDU API       │◀─────────│ Istio Ingress │                      │
│  │  (via Istio)    │          │ (JWT Policy)  │                      │
│  └────────┬────────┘          └───────────────┘                      │
│           │                                                          │
│           │ 4. Check Permissions                                     │
│           ▼                                                          │
│  ┌─────────────────┐                                                 │
│  │  Entitlements   │  ← Maps groups to data access                   │
│  │  Service        │                                                 │
│  └─────────────────┘                                                 │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## Identity Provider Selection

### Options Considered

| Option | Type | Pros | Cons |
|--------|------|------|------|
| **Keycloak** ✅ | Self-hosted | Full OIDC, OSDU-documented, admin UI, RBAC | Heavy (~1GB RAM) |
| **Authentik** | Self-hosted | Modern UI, Python-based | Less OSDU docs |
| **Dex** | Self-hosted | Lightweight, K8s-native | No admin UI, limited |
| **Auth0** | SaaS | Zero maintenance | Costs $$$, vendor lock-in |
| **Azure AD** | SaaS | Native OSDU support | Costs $$$, Azure lock-in |

### Decision

**Keycloak** — Most proven for self-hosted OSDU deployments, extensive documentation, full-featured admin console.

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                         OSDU Server                                │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                      Nginx (Edge)                            │  │
│  │                                                              │  │
│  │  Public (135.181.137.138):                                   │  │
│  │    osdu.domain.com      → Istio → OSDU APIs                  │  │
│  │    auth.domain.com      → Keycloak (login/token only)        │  │
│  │                                                              │  │
│  │  Private (10.10.0.1 - VPN):                                  │  │
│  │    keycloak.domain.com  → Keycloak Admin Console             │  │
│  │    (other admin UIs...)                                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     Kubernetes                               │  │
│  │                                                              │  │
│  │  ┌───────────────────┐                                       │  │
│  │  │     Keycloak      │                                       │  │
│  │  │                   │                                       │  │
│  │  │  ┌─────────────┐  │    ┌────────────────────────────────┐ │  │
│  │  │  │ Realm: osdu │  │    │      Istio Service Mesh        │ │  │
│  │  │  │             │  │    │                                │ │  │
│  │  │  │ - Users     │  │    │  RequestAuthentication         │ │  │
│  │  │  │ - Clients   │  │    │  (validates JWT from Keycloak) │ │  │
│  │  │  │ - Groups    │  │    │            │                   │ │  │
│  │  │  │ - Roles     │  │    │            ▼                   │ │  │
│  │  │  └─────────────┘  │    │  ┌──────────────────────────┐  │ │  │
│  │  └─────────┬─────────┘    │  │     OSDU Services        │  │ │  │
│  │            │              │  │  ┌────────┐ ┌──────────┐ │  │ │  │
│  │            ▼              │  │  │Storage │ │Entitlemnts│ │  │ │  │
│  │  ┌───────────────────┐    │  │  └────────┘ └──────────┘ │  │ │  │
│  │  │ PostgreSQL        │    │  │  ┌────────┐ ┌──────────┐ │  │ │  │
│  │  │ (Keycloak DB)     │    │  │  │ Search │ │  Legal   │ │  │ │  │
│  │  └───────────────────┘    │  │  └────────┘ └──────────┘ │  │ │  │
│  │                           │  └──────────────────────────┘  │ │  │
│  │                           └────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## Keycloak Configuration

### Realm Structure

```
Keycloak
└── Realm: osdu
    ├── Clients
    │   ├── osdu-api          (public, for user auth)
    │   ├── osdu-service      (confidential, for service-to-service)
    │   └── osdu-cli          (public, for CLI tools)
    │
    ├── Users
    │   ├── admin@osdu        (platform admin)
    │   ├── user1@osdu        (regular user)
    │   └── ...
    │
    ├── Groups (map to OSDU entitlements)
    │   ├── users.datalake.admins
    │   ├── users.datalake.viewers
    │   ├── users.datalake.editors
    │   ├── data.default.owners
    │   ├── data.default.viewers
    │   └── service.entitlements.admin
    │
    └── Roles
        ├── admin
        ├── user
        └── service
```

### Client Configuration

#### osdu-api (User Authentication)

```json
{
  "clientId": "osdu-api",
  "enabled": true,
  "publicClient": true,
  "standardFlowEnabled": true,
  "directAccessGrantsEnabled": true,
  "redirectUris": [
    "https://osdu.domain.com/*"
  ],
  "webOrigins": [
    "https://osdu.domain.com"
  ],
  "defaultClientScopes": [
    "openid",
    "profile",
    "email"
  ],
  "attributes": {
    "access.token.lifespan": "3600",
    "refresh.token.lifespan": "86400"
  }
}
```

#### osdu-service (Service-to-Service)

```json
{
  "clientId": "osdu-service",
  "enabled": true,
  "publicClient": false,
  "serviceAccountsEnabled": true,
  "standardFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "clientAuthenticatorType": "client-secret"
}
```

### Group to Entitlement Mapping

| Keycloak Group | OSDU Entitlement | Description |
|----------------|------------------|-------------|
| `users.datalake.admins` | `users.datalake.admins@osdu.domain.com` | Platform admins |
| `users.datalake.viewers` | `users.datalake.viewers@osdu.domain.com` | Read-only users |
| `users.datalake.editors` | `users.datalake.editors@osdu.domain.com` | Read-write users |
| `data.default.owners` | `data.default.owners@osdu.domain.com` | Data partition owners |
| `data.default.viewers` | `data.default.viewers@osdu.domain.com` | Data partition viewers |
| `service.entitlements.admin` | `service.entitlements.admin@osdu.domain.com` | Entitlements service admin |

## Istio JWT Validation

### RequestAuthentication

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: osdu-jwt-auth
  namespace: osdu-core
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: osdu
  jwtRules:
    - issuer: "https://auth.domain.com/realms/osdu"
      jwksUri: "https://auth.domain.com/realms/osdu/protocol/openid-connect/certs"
      audiences:
        - "osdu-api"
      forwardOriginalToken: true
```

### AuthorizationPolicy (Require Valid JWT)

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: osdu-require-jwt
  namespace: osdu-core
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: osdu
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
    # Allow health checks without auth
    - to:
        - operation:
            paths: ["/health", "/ready", "/liveness"]
```

## Token Structure

### Access Token (JWT)

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id-from-keycloak"
  },
  "payload": {
    "iss": "https://auth.domain.com/realms/osdu",
    "sub": "user-uuid",
    "aud": "osdu-api",
    "exp": 1708150800,
    "iat": 1708147200,
    "azp": "osdu-api",
    "scope": "openid profile email",
    "email": "user@example.com",
    "preferred_username": "user1",
    "groups": [
      "users.datalake.viewers",
      "data.default.viewers"
    ]
  }
}
```

### Token Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /realms/osdu/.well-known/openid-configuration` | Discovery document |
| `POST /realms/osdu/protocol/openid-connect/token` | Get access token |
| `GET /realms/osdu/protocol/openid-connect/certs` | JWKS (public keys) |
| `POST /realms/osdu/protocol/openid-connect/logout` | Logout |
| `GET /realms/osdu/protocol/openid-connect/userinfo` | User info |

## Nginx Configuration

### Public Auth Endpoint

```nginx
# /etc/nginx/sites-available/osdu-public.conf

# Keycloak - Public endpoints only
server {
    listen 135.181.137.138:443 ssl http2;
    server_name auth.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    # Allow: OpenID Connect endpoints
    location /realms/osdu/protocol/openid-connect/ {
        proxy_pass http://127.0.0.1:30800;
        include snippets/proxy-params.conf;
    }

    # Allow: Well-known discovery
    location /realms/osdu/.well-known/ {
        proxy_pass http://127.0.0.1:30800;
        include snippets/proxy-params.conf;
    }

    # Allow: Login pages and resources
    location /realms/osdu/login-actions/ {
        proxy_pass http://127.0.0.1:30800;
        include snippets/proxy-params.conf;
    }

    location /realms/osdu/resources/ {
        proxy_pass http://127.0.0.1:30800;
        include snippets/proxy-params.conf;
    }

    # Block: Everything else (admin, other realms)
    location / {
        return 403 "Access denied";
    }
}
```

### Private Admin Console

```nginx
# /etc/nginx/sites-available/osdu-private.conf

# Keycloak Admin - VPN only
server {
    listen 10.10.0.1:443 ssl http2;
    server_name keycloak.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    # Full access to Keycloak (admin included)
    location / {
        proxy_pass http://127.0.0.1:30800;
        include snippets/proxy-params.conf;

        # WebSocket support for admin console
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Service Accounts

### For OSDU Services

Each OSDU service that needs to call other services uses a service account:

```yaml
# Service account for Storage service
apiVersion: v1
kind: Secret
metadata:
  name: storage-service-credentials
  namespace: osdu-core
type: Opaque
stringData:
  client-id: "osdu-service"
  client-secret: "<secret-from-keycloak>"
```

### Token Exchange (Service-to-Service)

```bash
# Get service account token
curl -X POST "https://auth.domain.com/realms/osdu/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=osdu-service" \
  -d "client_secret=<secret>"
```

## OSDU Entitlements Integration

### Bootstrap Entitlements

Initial groups must be created in the Entitlements service:

```bash
# Create initial entitlements (via OSDU API)
curl -X POST "https://osdu.domain.com/api/entitlements/v2/groups" \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -H "data-partition-id: osdu" \
  -d '{
    "name": "users.datalake.admins",
    "description": "Platform administrators"
  }'
```

### User Provisioning Flow

```
1. Admin creates user in Keycloak
2. Admin assigns Keycloak groups to user
3. User logs in → Gets JWT with groups claim
4. OSDU services read groups from JWT
5. Entitlements service validates group membership
6. Access granted based on entitlements
```

## Deployment

### Keycloak Kubernetes Resources

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: auth
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:23.0
          args: ["start"]
          env:
            - name: KEYCLOAK_ADMIN
              value: "admin"
            - name: KEYCLOAK_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-secrets
                  key: admin-password
            - name: KC_DB
              value: "postgres"
            - name: KC_DB_URL
              value: "jdbc:postgresql://keycloak-postgres:5432/keycloak"
            - name: KC_DB_USERNAME
              value: "keycloak"
            - name: KC_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-secrets
                  key: db-password
            - name: KC_HOSTNAME
              value: "auth.domain.com"
            - name: KC_PROXY
              value: "edge"
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: auth
spec:
  type: NodePort
  selector:
    app: keycloak
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30800
```

### Resource Requirements

| Component | Memory | CPU | Storage |
|-----------|--------|-----|---------|
| Keycloak | 512Mi-1Gi | 0.5-1 | - |
| PostgreSQL (Keycloak) | 256Mi-512Mi | 0.25 | 5Gi |
| **Total** | ~1.5Gi | ~1.25 | 5Gi |

## Security Checklist

### Keycloak Hardening

- [ ] Change default admin password
- [ ] Enable brute force protection
- [ ] Configure password policies
- [ ] Limit token lifespans
- [ ] Enable audit logging
- [ ] Restrict admin console to VPN

### Token Security

- [ ] Use short-lived access tokens (1 hour)
- [ ] Use refresh tokens for long sessions
- [ ] Validate audience claim in services
- [ ] Rotate signing keys periodically

### Network Security

- [ ] Admin console only via VPN
- [ ] Token endpoints rate-limited
- [ ] HTTPS everywhere

## Testing Authentication

### Get Token (Password Grant)

```bash
# Get user token
TOKEN=$(curl -s -X POST "https://auth.domain.com/realms/osdu/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=osdu-api" \
  -d "username=testuser" \
  -d "password=testpass" \
  | jq -r '.access_token')

echo $TOKEN
```

### Test OSDU API

```bash
# Call OSDU API with token
curl -X GET "https://osdu.domain.com/api/partition/v1/partitions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json"
```

### Decode Token (Debug)

```bash
# Decode JWT payload (without verification)
echo $TOKEN | cut -d'.' -f2 | base64 -d | jq .
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 401 Unauthorized | Invalid/expired token | Check token expiry, refresh |
| 403 Forbidden | Missing entitlements | Add user to required groups |
| Token validation failed | Wrong issuer/audience | Check Istio RequestAuthentication config |
| Can't reach Keycloak | DNS/network issue | Verify auth.domain.com resolves |

### Debug Commands

```bash
# Check Keycloak pods
kubectl get pods -n auth

# Check Keycloak logs
kubectl logs -n auth deployment/keycloak

# Test JWKS endpoint
curl https://auth.domain.com/realms/osdu/protocol/openid-connect/certs

# Check Istio auth policy
kubectl get requestauthentication -n osdu-core
kubectl get authorizationpolicy -n osdu-core
```
