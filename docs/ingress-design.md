# Ingress Design

## Overview

This document details the ingress architecture using Nginx as an edge proxy in front of Istio and other Kubernetes services.

## Architecture Layers

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  Nginx (Host - systemd)         │  ← Layer 1: Edge Proxy
│  - SSL termination              │
│  - Subdomain routing            │
│  - Rate limiting                │
│  Ports: 80, 443                 │
└─────────────┬───────────────────┘
              │
    ┌─────────┴─────────┬─────────────────┐
    │                   │                 │
    ▼                   ▼                 ▼
┌─────────┐      ┌───────────┐     ┌───────────┐
│ Istio   │      │  MinIO    │     │  Kibana   │
│ Ingress │      │  :30900   │     │  :30561   │
│ :30080  │      └───────────┘     └───────────┘
└────┬────┘           ↑                  ↑
     │                │                  │
     ▼                └──────────────────┘
┌─────────────┐              │
│ Istio Mesh  │      Kubernetes Cluster
│ (OSDU SVCs) │      (NodePort Services)
└─────────────┘
```

## Nginx Configuration

### Directory Structure

```
/etc/nginx/
├── nginx.conf
├── sites-available/
│   └── osdu-proxy.conf
├── sites-enabled/
│   └── osdu-proxy.conf -> ../sites-available/osdu-proxy.conf
├── snippets/
│   ├── ssl-params.conf
│   └── proxy-params.conf
```

### Main Proxy Configuration

```nginx
# /etc/nginx/sites-available/osdu-proxy.conf

# ============================================
# SSL Parameters (shared)
# ============================================
# Include in each server block or use snippets

# ============================================
# HTTP → HTTPS Redirect
# ============================================
server {
    listen 80;
    listen [::]:80;
    server_name *.osdu.yourdomain.com;
    return 301 https://$host$request_uri;
}

# ============================================
# OSDU Platform (via Istio Ingress)
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name osdu.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/osdu.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/osdu.yourdomain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://127.0.0.1:30080;
        include snippets/proxy-params.conf;
    }
}

# ============================================
# MinIO API
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name minio.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/osdu.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/osdu.yourdomain.com/privkey.pem;
    include snippets/ssl-params.conf;

    # MinIO needs special handling for large uploads
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://127.0.0.1:30900;
        include snippets/proxy-params.conf;
        
        # S3 signature compatibility
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        
        proxy_connect_timeout 300;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;
    }
}

# ============================================
# MinIO Console
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name console.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/osdu.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/osdu.yourdomain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://127.0.0.1:30901;
        include snippets/proxy-params.conf;
        
        # WebSocket support for console
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ============================================
# Kibana
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name kibana.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/osdu.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/osdu.yourdomain.com/privkey.pem;
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
# Grafana
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name grafana.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/osdu.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/osdu.yourdomain.com/privkey.pem;
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
# ArgoCD
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name argocd.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/osdu.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/osdu.yourdomain.com/privkey.pem;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass https://127.0.0.1:30443;
        include snippets/proxy-params.conf;
        
        # ArgoCD uses gRPC
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### SSL Parameters Snippet

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

# Security headers
add_header X-Frame-Options DENY always;
add_header X-Content-Type-Options nosniff always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=63072000" always;
```

### Proxy Parameters Snippet

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

## SSL Certificate Setup

### Option 1: Wildcard Cert with Cloudflare DNS (Recommended)

```bash
# Install certbot and cloudflare plugin
apt install certbot python3-certbot-dns-cloudflare

# Create cloudflare credentials file
cat > /etc/letsencrypt/cloudflare.ini << EOF
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
EOF
chmod 600 /etc/letsencrypt/cloudflare.ini

# Get wildcard certificate
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.osdu.yourdomain.com" \
  -d "osdu.yourdomain.com"

# Auto-renewal is set up automatically
```

### Option 2: Individual Certs with HTTP Challenge

```bash
# For each subdomain
certbot --nginx -d osdu.yourdomain.com
certbot --nginx -d minio.yourdomain.com
certbot --nginx -d kibana.yourdomain.com
# ... etc
```

## Kubernetes Service Exposure

Services are exposed via NodePort to be accessible from the host Nginx:

```yaml
# Example: Expose Istio Ingress Gateway
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
spec:
  type: NodePort
  ports:
    - name: http2
      port: 80
      targetPort: 8080
      nodePort: 30080
    - name: https
      port: 443
      targetPort: 8443
      nodePort: 30443
```

### Port Allocation Plan

| Service | NodePort | Internal Port |
|---------|----------|---------------|
| Istio Ingress (HTTP) | 30080 | 8080 |
| Istio Ingress (HTTPS) | 30443 | 8443 |
| MinIO API | 30900 | 9000 |
| MinIO Console | 30901 | 9001 |
| Kibana | 30561 | 5601 |
| Grafana | 30300 | 3000 |
| Prometheus | 30909 | 9090 |
| ArgoCD | 30443 | 443 |

## Security Considerations

1. **Firewall**: Only expose ports 80, 443 externally
2. **NodePorts**: Only accessible from localhost (127.0.0.1)
3. **SSL**: Enforce HTTPS with HSTS
4. **Rate Limiting**: Add nginx rate limiting for public endpoints
5. **Auth**: Add basic auth or OAuth proxy for admin UIs (see Authentication Design)
