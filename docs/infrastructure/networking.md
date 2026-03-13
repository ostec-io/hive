# Networking Reference

## Overview

HIVE uses multiple network layers for security and isolation:

1. **WireGuard VPN** - Encrypted mesh between all nodes
2. **Docker Overlay Networks** - Internal service communication
3. **Public Network** - Internet-facing endpoints (Frontend only)

## Port Reference

### HIVE Components

| Component | Port | Protocol | Purpose |
|-----------|------|----------|---------|
| hive-platform | 9130 | HTTP/HTTPS | REST API |
| hive-platform | 8443 | WSS | Agent WebSocket connections |
| hive-gateway | 9131 | WSS | Browser WebSocket connections |
| hive-webapp | 3000 | HTTP | Development server |
| hive-webapp | 80/443 | HTTP/HTTPS | Production (via Traefik) |
| hive-agent | 9190 | HTTP | Prometheus metrics |

### Data Services

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| MongoDB | 27017 | TCP | Database connections |
| Neo4j HTTP | 7474 | HTTP | Browser interface |
| Neo4j Bolt | 7687 | TCP | Database connections |
| Elasticsearch | 9200 | HTTP | REST API |
| Elasticsearch | 9300 | TCP | Inter-node communication |
| RabbitMQ | 5672 | AMQP | Message broker |
| RabbitMQ | 15672 | HTTP | Management UI |
| Keycloak | 8080 | HTTP | Auth server |

### Infrastructure

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| WireGuard | 51820 | UDP | VPN tunnel |
| Docker Swarm | 2377 | TCP | Cluster management |
| Docker Swarm | 7946 | TCP/UDP | Node discovery |
| Docker Swarm | 4789 | UDP | Overlay network (VXLAN) |
| Traefik | 80 | HTTP | Web traffic |
| Traefik | 443 | HTTPS | Secure web traffic |
| Traefik | 8080 | HTTP | Dashboard (internal) |

## Network Topology

### Production (Dual Swarm)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                     │
│                            │                                         │
│                     ┌──────┴──────┐                                  │
│                     │   80/443    │                                  │
│                     └──────┬──────┘                                  │
├─────────────────────────────┼───────────────────────────────────────┤
│  FRONTEND SWARM            │                                        │
│  ┌─────────────────────────┴─────────────────────────────────────┐  │
│  │  Traefik (10.0.0.20)                                          │  │
│  │     │                                                         │  │
│  │     ├──► hive-webapp (:3000)                                  │  │
│  │     └──► hive-gateway (:9131)                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                            │                                        │
│                     ┌──────┴──────┐                                  │
│                     │  VPN ONLY   │                                  │
│                     │   HTTPS     │                                  │
│                     └──────┬──────┘                                  │
├─────────────────────────────┼───────────────────────────────────────┤
│  API/INFRA SWARM           │                                        │
│  ┌─────────────────────────┴─────────────────────────────────────┐  │
│  │  Traefik (10.0.0.1)                                           │  │
│  │     │                                                         │  │
│  │     ├──► hive-platform (:9130, :8443)                         │  │
│  │     │                                                         │  │
│  │     └──► hive-agent (connects to platform)                    │  │
│  │                                                               │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  OVERLAY NETWORK (encrypted)                             │  │  │
│  │  │     MongoDB, Neo4j, Elasticsearch, RabbitMQ, Keycloak   │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Development (Single Swarm)

```
┌─────────────────────────────────────────────────────────────────────┐
│  DEVELOPMENT SWARM (localhost)                                       │
│                                                                      │
│  Browser ──► localhost:3000 (webapp)                                │
│         ──► localhost:9131 (gateway)                                │
│                                                                      │
│  hive-webapp (:3000)                                                │
│       │                                                             │
│       └──► hive-platform (:9130)                                    │
│                   │                                                 │
│                   ├──► MongoDB (:27017)                             │
│                   ├──► Keycloak (:8080)                             │
│                   └──► hive-agent ──► Docker API                    │
│                                                                      │
│  hive-gateway (:9131)                                               │
│       └──► hive-platform (internal)                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Firewall Rules

### Frontend Servers (Public-Facing)

```bash
# UFW Configuration

# SSH - restrict to known IPs
ufw allow from <admin_ip> to any port 22

# Web traffic - public
ufw allow 80/tcp
ufw allow 443/tcp

# WireGuard VPN
ufw allow 51820/udp

# Docker Swarm - VPN only
ufw allow from 10.0.0.0/24 to any port 2377 proto tcp
ufw allow from 10.0.0.0/24 to any port 7946
ufw allow from 10.0.0.0/24 to any port 4789 proto udp

# Default deny
ufw default deny incoming
ufw enable
```

### API/Infrastructure Servers (VPN-Only)

```bash
# UFW Configuration

# SSH - VPN only
ufw allow from 10.0.0.0/24 to any port 22

# WireGuard VPN
ufw allow 51820/udp

# All VPN traffic allowed
ufw allow from 10.0.0.0/24

# Default deny all public
ufw default deny incoming
ufw enable
```

## Docker Networks

### Creating Networks

```bash
# Main infrastructure overlay (encrypted)
docker network create \
  --driver overlay \
  --attachable \
  --opt encrypted \
  hive-network

# Public-facing (for Traefik routing)
docker network create \
  --driver overlay \
  --attachable \
  traefik-public
```

### Network Inspection

```bash
# List all networks
docker network ls

# Inspect a network
docker network inspect hive-network

# Check which services are connected
docker network inspect hive-network --format '{{ range .Containers }}{{ .Name }} {{ end }}'
```

## DNS Resolution

### Internal (Docker Swarm DNS)

Services can reach each other by name within the same overlay network:

| DNS Name | Resolves To |
|----------|-------------|
| `mongo-1` | MongoDB primary container |
| `elasticsearch-1` | Elasticsearch node 1 |
| `keycloak` | Keycloak service (load balanced) |
| `hive-platform` | Platform service |

### External (VPN DNS via CoreDNS)

For admin access from VPN clients, CoreDNS provides resolution:

| DNS Name | Resolves To |
|----------|-------------|
| `auth.vpn.example.com` | 10.0.0.1 (Traefik → Keycloak) |
| `api.vpn.example.com` | 10.0.0.1 (Traefik → Platform) |
| `rabbitmq.vpn.example.com` | 10.0.0.1 (Traefik → RabbitMQ UI) |

## Connection Flows

### Browser → Webapp → Platform

```
Browser
  │
  ├─ HTTPS :443 ─► Traefik (Frontend)
  │                    │
  │                    └─► hive-webapp (:3000)
  │                            │
  │                            └─ HTTPS via VPN ─► Traefik (API)
  │                                                    │
  │                                                    └─► hive-platform (:9130)
  │
  └─ WSS :443 ──► hive-gateway (:9131)
                       │
                       └─► hive-platform (internal)
```

### API/MCP → Platform

```
API/MCP Client
  │
  ├─ HTTPS :443 ─► Traefik (Frontend)
  │                    │
  │                    └─► Traefik (API) ─► hive-platform (:9130)
  │
  └─ (VPN‑only optional) ─► Traefik (API) ─► hive-platform (:9130)
```

### Agent → Platform

```
hive-agent (on any cluster)
  │
  └─ WSS :8443 (mTLS) ─► hive-platform
                              │
                              └─ Ed25519 signed jobs
```

### Platform → Data Services

```
hive-platform
  │
  ├─ mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017
  │
  ├─ bolt://neo4j:7687
  │
  ├─ http://elasticsearch-1:9200
  │
  ├─ amqp://rabbitmq-1:5672
  │
  └─ http://keycloak:8080 (JWT validation)
```

## Troubleshooting

### Test Network Connectivity

```bash
# From a container, test DNS resolution
docker exec <container> nslookup mongo-1

# Test TCP connectivity
docker exec <container> nc -zv mongo-1 27017

# Test HTTP endpoint
docker exec <container> curl -I http://keycloak:8080/health/ready
```

### Check Swarm Routing Mesh

```bash
# View published ports
docker service inspect <service> --format '{{ .Endpoint.Ports }}'

# Check service VIP
docker service inspect <service> --format '{{ .Endpoint.VirtualIPs }}'
```

### Network Debug Container

```bash
# Run a debug container on the network
docker run -it --rm --network hive-network nicolaka/netshoot

# Inside the container:
dig mongo-1
ping elasticsearch-1
curl http://keycloak:8080
```

## See Also

- [VPN Setup](vpn-setup.md) - WireGuard configuration
- [Swarm Setup](swarm-setup.md) - Docker Swarm deployment
- [Security Architecture](security.md) - Network isolation model
- [Production Environment](../environments/production.md) - Dual-swarm setup
