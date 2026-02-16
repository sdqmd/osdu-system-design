# Ingress Design

## Overview

This document details the ingress architecture using Nginx as an edge proxy with split public/private listeners.

## Architecture

```
                         INTERNET
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │  Public Access  │           │   VPN Access    │
    │ 135.181.137.138 │           │   10.10.0.1     │
    │   Ports 80,443  │           │   (WireGuard)   │
    └────────┬────────┘           └────────┬────────┘
             │                             │
             ▼                             ▼
    ┌─────────────────────────────────────────────────┐
    │              Nginx (systemd, host)              │
    │                                                 │
    │  ┌─────────────────┐    ┌─────────────────────┐ │
    │  │ Public Listener │    │  Private Listeners  │ │
    │  │                 │    │                     │ │
    │  │ osdu.*          │    │ minio.*   kibana.*  │ │
    │  │    ↓            │    │ console.* grafana.* │ │
    │  │ Istio :30080    │    │ argocd.*  kiali.*   │ │
    │  └─────────────────┘    │ jaeger.*            │ │
    │                         │    ↓                │ │
    │                         │ Internal Services   │ │
    │                         └─────────────────────┘ │
    └─────────────────────────────────────────────────┘
```

## Directory Structure

```
/etc/nginx/
├── nginx.conf
├── sites-available/
│   ├── osdu-public.conf      # Public services (OSDU API)
│   └── osdu-private.conf     # Private services (VPN only)
├── sites-enabled/
│   ├── osdu-public.conf -> ../sites-available/osdu-public.conf
│   └── osdu-private.conf -> ../sites-available/osdu-private.conf
└── snippets/
    ├── ssl-params.conf
    └── proxy-params.conf
```

## Public Services Configuration

Only OSDU API is exposed publicly.

```nginx
# /etc/nginx/sites-available/osdu-public.conf

# ============================================
# HTTP → HTTPS Redirect (Public)
# ============================================
server {
    listen 135.181.137.138:80;
    listen [::]:80;
    server_name osdu.domain.com;
    return 301 https://$host$request_uri;
}

# ============================================
# OSDU Platform API (Public)
# ============================================
server {
    listen 135.181.137.138:443 ssl http2;
    server_name osdu.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    # Security headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Rate limiting (optional)
    limit_req zone=api burst=20 nodelay;

    location / {
        proxy_pass http://127.0.0.1:30080;
        include snippets/proxy-params.conf;

        # Timeout settings for API calls
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }

    # Health check endpoint (no auth)
    location /health {
        proxy_pass http://127.0.0.1:30080/health;
        access_log off;
    }
}

# ============================================
# Rate Limiting Zone (define in nginx.conf http block)
# ============================================
# limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
```

## Private Services Configuration

All admin and internal services bound to VPN IP only.

```nginx
# /etc/nginx/sites-available/osdu-private.conf

# ============================================
# HTTP → HTTPS Redirect (Private/VPN)
# ============================================
server {
    listen 10.10.0.1:80;
    server_name *.domain.com;
    return 301 https://$host$request_uri;
}

# ============================================
# MinIO API (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name minio.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    # MinIO needs special handling
    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://127.0.0.1:30900;
        include snippets/proxy-params.conf;

        proxy_connect_timeout 300;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;
    }
}

# ============================================
# MinIO Console (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name console.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_buffering off;

    location / {
        proxy_pass http://127.0.0.1:30901;
        include snippets/proxy-params.conf;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ============================================
# Kibana (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name kibana.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://127.0.0.1:30561;
        include snippets/proxy-params.conf;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ============================================
# Grafana (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name grafana.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://127.0.0.1:30300;
        include snippets/proxy-params.conf;

        # WebSocket support for live updates
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ============================================
# ArgoCD (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name argocd.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass https://127.0.0.1:30443;
        include snippets/proxy-params.conf;

        # gRPC and WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # ArgoCD uses gRPC-Web
        grpc_read_timeout 300s;
        grpc_send_timeout 300s;
    }
}

# ============================================
# Kiali (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name kiali.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://127.0.0.1:30686;
        include snippets/proxy-params.conf;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ============================================
# Jaeger (Private)
# ============================================
server {
    listen 10.10.0.1:443 ssl http2;
    server_name jaeger.domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://127.0.0.1:30686;
        include snippets/proxy-params.conf;
    }
}
```

## Shared Snippets

### SSL Parameters

```nginx
# /etc/nginx/snippets/ssl-params.conf

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_ecdh_curve secp384r1;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;
```

### Proxy Parameters

```nginx
# /etc/nginx/snippets/proxy-params.conf

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;

proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

## Port Allocation

### Public Ports (Firewall Open)

| Port | Protocol | Service |
|------|----------|---------|
| 80 | TCP | HTTP redirect |
| 443 | TCP | HTTPS (OSDU only) |
| 51820 | UDP | WireGuard VPN |

### NodePort Allocation (Internal)

| Service | NodePort | Internal Port |
|---------|----------|---------------|
| Istio Ingress (HTTP) | 30080 | 8080 |
| Istio Ingress (HTTPS) | 30443 | 8443 |
| MinIO API | 30900 | 9000 |
| MinIO Console | 30901 | 9001 |
| Kibana | 30561 | 5601 |
| Grafana | 30300 | 3000 |
| Prometheus | 30909 | 9090 |
| ArgoCD Server | 30443 | 443 |
| Kiali | 30686 | 20001 |
| Jaeger UI | 30686 | 16686 |

## SSL Certificate Setup

### Wildcard Certificate with Cloudflare DNS

```bash
# Install certbot and cloudflare plugin
apt install certbot python3-certbot-dns-cloudflare

# Create cloudflare credentials
cat > /etc/letsencrypt/cloudflare.ini << EOF
dns_cloudflare_api_token = YOUR_API_TOKEN
EOF
chmod 600 /etc/letsencrypt/cloudflare.ini

# Get wildcard certificate
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "domain.com" \
  -d "*.domain.com"

# Verify
certbot certificates
```

### Auto-Renewal

```bash
# Test renewal
certbot renew --dry-run

# Renewal hook to reload nginx
cat > /etc/letsencrypt/renewal-hooks/post/nginx-reload.sh << 'EOF'
#!/bin/bash
systemctl reload nginx
EOF
chmod +x /etc/letsencrypt/renewal-hooks/post/nginx-reload.sh
```

## Testing

### Verify Public Access

```bash
# From external machine (not on VPN)
curl -I https://osdu.domain.com/api/health

# Should work
# Other subdomains should NOT be reachable
curl -I https://kibana.domain.com  # Should fail/timeout
```

### Verify Private Access

```bash
# Connect to VPN first
wg-quick up wg0

# Test internal services
curl -I https://kibana.domain.com
curl -I https://grafana.domain.com
curl -I https://console.domain.com

# All should work
```

## Security Notes

1. **IP Binding is Critical**: Always use explicit IP binding (`listen 10.10.0.1:443`) for private services, never `listen 443`

2. **Defense in Depth**: Even with IP binding, firewall rules should also restrict access

3. **VPN Required**: Users must be connected to WireGuard VPN to access private services

4. **Same SSL Cert**: Wildcard cert works for all subdomains regardless of which IP serves them
