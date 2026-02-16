# Network Security Design

## Overview

This document details the network security architecture, including VPN configuration, firewall rules, and network segmentation.

## Network Topology

```
                            INTERNET
                                │
                                │
                    ┌───────────┴───────────┐
                    │    Hetzner Network    │
                    │                       │
                    └───────────┬───────────┘
                                │
                                │
┌───────────────────────────────┴────────────────────────────────────┐
│                         OSDU SERVER                                │
│                      135.181.137.138                               │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Network Interfaces                        │  │
│  │                                                              │  │
│  │  eth0 (Public):        135.181.137.138                       │  │
│  │    ├── Port 80/tcp     HTTP redirect                         │  │
│  │    ├── Port 443/tcp    HTTPS (osdu.* only)                   │  │
│  │    └── Port 51820/udp  WireGuard VPN                         │  │
│  │                                                              │  │
│  │  wg0 (WireGuard VPN):  10.10.0.1/24                          │  │
│  │    └── All ports       Internal services                     │  │
│  │                                                              │  │
│  │  docker0/cni0:         172.17.0.0/16 (Kubernetes pods)       │  │
│  │                                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    VPN Client Pool                           │  │
│  │                                                              │  │
│  │  10.10.0.1    Server (gateway)                               │  │
│  │  10.10.0.2    Admin laptop                                   │  │
│  │  10.10.0.3    Admin phone                                    │  │
│  │  10.10.0.4    Dev workstation                                │  │
│  │  10.10.0.5+   Future clients                                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## WireGuard VPN Configuration

### Server Configuration

```ini
# /etc/wireguard/wg0.conf

[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

# Optional: Run commands after interface up/down
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Peer: Admin Laptop
[Peer]
PublicKey = <CLIENT_1_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32

# Peer: Admin Phone
[Peer]
PublicKey = <CLIENT_2_PUBLIC_KEY>
AllowedIPs = 10.10.0.3/32

# Peer: Dev Workstation
[Peer]
PublicKey = <CLIENT_3_PUBLIC_KEY>
AllowedIPs = 10.10.0.4/32
```

### Client Configuration Template

```ini
# Client: Admin Laptop (10.10.0.2)

[Interface]
Address = 10.10.0.2/24
PrivateKey = <CLIENT_PRIVATE_KEY>
DNS = 10.10.0.1  # Use server for DNS (optional)

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = 135.181.137.138:51820
AllowedIPs = 10.10.0.0/24  # Only route VPN subnet
PersistentKeepalive = 25   # Keep connection alive behind NAT
```

### Key Generation Commands

```bash
# Generate server keys
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key

# Generate client keys (do this for each client)
wg genkey | tee client1_private.key | wg pubkey > client1_public.key
```

### Service Management

```bash
# Enable and start WireGuard
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Check status
wg show

# Add peer without restart
wg set wg0 peer <PUBLIC_KEY> allowed-ips 10.10.0.X/32
```

## Firewall Configuration (UFW)

### Rules

```bash
# Reset to defaults
ufw default deny incoming
ufw default allow outgoing

# Allow SSH (be careful - don't lock yourself out!)
ufw allow 22/tcp

# Public services
ufw allow 80/tcp    # HTTP redirect
ufw allow 443/tcp   # HTTPS

# WireGuard VPN
ufw allow 51820/udp

# Allow all traffic from VPN interface
ufw allow in on wg0

# Enable firewall
ufw enable
```

### Verification

```bash
# Check status
ufw status verbose

# Expected output:
# To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW       Anywhere
# 80/tcp                     ALLOW       Anywhere
# 443/tcp                    ALLOW       Anywhere
# 51820/udp                  ALLOW       Anywhere
# Anywhere on wg0            ALLOW       Anywhere
```

## Nginx Network Binding

### Split Listener Configuration

```nginx
# Public listener - binds to public IP only
server {
    listen 135.181.137.138:443 ssl http2;
    server_name osdu.domain.com;
    # ... OSDU config
}

# Private listeners - bind to VPN IP only
server {
    listen 10.10.0.1:443 ssl http2;
    server_name minio.domain.com;
    # ... MinIO config
}

server {
    listen 10.10.0.1:443 ssl http2;
    server_name console.domain.com;
    # ... MinIO Console config
}

server {
    listen 10.10.0.1:443 ssl http2;
    server_name kibana.domain.com;
    # ... Kibana config
}

# ... etc for other internal services
```

### Why Explicit IP Binding?

- `listen 443` → Binds to ALL interfaces (0.0.0.0)
- `listen 135.181.137.138:443` → Binds to public IP only
- `listen 10.10.0.1:443` → Binds to VPN IP only

This ensures internal services are **not accessible** from the public internet, even if firewall rules were misconfigured.

## Private DNS

### Option A: dnsmasq on Server

```bash
# Install dnsmasq
apt install dnsmasq

# Configure /etc/dnsmasq.d/osdu-internal.conf
listen-address=10.10.0.1
bind-interfaces

# Internal service records
address=/minio.domain.com/10.10.0.1
address=/console.domain.com/10.10.0.1
address=/kibana.domain.com/10.10.0.1
address=/grafana.domain.com/10.10.0.1
address=/argocd.domain.com/10.10.0.1
address=/kiali.domain.com/10.10.0.1
address=/jaeger.domain.com/10.10.0.1

# Forward other queries to upstream DNS
server=1.1.1.1
server=8.8.8.8
```

WireGuard clients use `DNS = 10.10.0.1` to resolve internal names.

### Option B: Client Hosts File

For simpler setups, add to each client's hosts file:

```
# Linux/Mac: /etc/hosts
# Windows: C:\Windows\System32\drivers\etc\hosts

10.10.0.1  minio.domain.com
10.10.0.1  console.domain.com
10.10.0.1  kibana.domain.com
10.10.0.1  grafana.domain.com
10.10.0.1  argocd.domain.com
10.10.0.1  kiali.domain.com
10.10.0.1  jaeger.domain.com
```

## SSL Certificates for Private Services

Private services still need valid SSL certificates. Options:

### Option 1: Same Wildcard Cert (Recommended)

Use the same `*.domain.com` wildcard cert for both public and private services. The cert is valid regardless of which IP serves it.

### Option 2: Separate Internal CA

For air-gapped or extra security:
1. Create internal CA
2. Generate certs for internal services
3. Install CA cert on all VPN clients

More complex, usually not necessary for personal/learning setups.

## Security Checklist

### Server Hardening

- [ ] SSH key-only authentication (disable password)
- [ ] Fail2ban installed and configured
- [ ] Automatic security updates enabled
- [ ] Non-root user for daily operations
- [ ] UFW firewall enabled

### Network Security

- [ ] WireGuard configured and running
- [ ] Only ports 22, 80, 443, 51820 exposed publicly
- [ ] Nginx private services bound to VPN IP only
- [ ] Private DNS configured for internal resolution

### Access Control

- [ ] VPN keys generated for each user/device
- [ ] Unused VPN peers removed promptly
- [ ] Admin UI passwords set (Grafana, ArgoCD, etc.)
- [ ] OSDU authentication configured

## Monitoring & Alerts

### VPN Monitoring

```bash
# Check connected peers
wg show wg0

# Watch for connections
watch -n 5 'wg show wg0'
```

### Connection Logging

```bash
# Add to wg0.conf PostUp
PostUp = iptables -A INPUT -i wg0 -j LOG --log-prefix "WG-IN: "

# View logs
journalctl -f | grep "WG-IN"
```

## Troubleshooting

### VPN Connection Issues

```bash
# Server side - check interface
ip addr show wg0
wg show wg0

# Check if port is open
ss -ulnp | grep 51820

# Client side - check handshake
wg show

# If "latest handshake" is missing, check:
# 1. Server endpoint and port
# 2. Public key matches
# 3. Firewall allows UDP 51820
```

### Cannot Access Internal Services

```bash
# 1. Check VPN is connected
ping 10.10.0.1

# 2. Check DNS resolution
nslookup kibana.domain.com

# 3. Check Nginx is listening
curl -v https://10.10.0.1:443 --resolve kibana.domain.com:443:10.10.0.1

# 4. Check service is running
curl http://127.0.0.1:30561  # Direct to NodePort
```
