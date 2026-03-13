# Security Architecture

## Overview

HIVE's infrastructure follows a **zero-trust security model** where nothing is trusted by default. Every connection must be authenticated, encrypted, and authorized.

## Security Principles

### 1. Zero-Trust Network

**Nothing is trusted by default.** Every connection must be authenticated.

- All inter-node communication happens over **WireGuard VPN**
- Each node has a unique cryptographic identity (public/private keypair)
- No services exposed on public IPs except:
  - Frontend: 80/443 (web traffic)
  - All nodes: 51820/UDP (WireGuard)

### 2. Defense in Depth

**Multiple layers of security**, so compromising one doesn't compromise all.

| Layer | Protection |
|-------|------------|
| Network | WireGuard encryption (ChaCha20-Poly1305) |
| Firewall | UFW with explicit allow rules only |
| Application | Traefik with TLS termination |
| Database | Overlay network isolation, no VPN exposure |
| Access | SSH key authentication only |
| Agent Auth | mTLS + Ed25519 message signatures |
| User Auth | JWT via Keycloak with tenant isolation |

### 3. Least Privilege

**Services only have access to what they need.**

- Frontend servers can only reach the API via HTTPS (port 443)
- API workers connect to databases via Docker overlay network
- Databases are **never exposed** to the VPN — only accessible via internal overlay DNS
- Node labels enforce where workloads can run:
  - `role=infra` → databases, message queues
  - `role=api` → application services
  - `role=frontend` → web applications

## Network Isolation

### Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────┐
│ PUBLIC ZONE                                             │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Frontend Swarm (10.0.0.20-29)                       │ │
│ │ • Exposed: 80/443                                   │ │
│ │ • Serves webapp, gateway                            │ │
│ │ • Can ONLY reach API tier via HTTPS                 │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                           │ HTTPS only
                           ▼
┌─────────────────────────────────────────────────────────┐
│ APPLICATION ZONE (VPN Only)                             │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ API Workers (10.0.0.10-19)                          │ │
│ │ • No public exposure                                │ │
│ │ • Accessed via Traefik reverse proxy                │ │
│ │ • Connects to databases via overlay network         │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                           │ Overlay network
                           ▼
┌─────────────────────────────────────────────────────────┐
│ DATA ZONE (Overlay Only)                                │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Infrastructure Services (10.0.0.1-9)                │ │
│ │ • MongoDB, Elasticsearch, Neo4j, RabbitMQ           │ │
│ │ • NOT accessible via VPN                            │ │
│ │ • Only reachable via Docker overlay DNS             │ │
│ │ • Encrypted overlay network (IPSec)                 │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Attack Surface Analysis

**If an attacker compromises a frontend server:**
- They're on the VPN, but...
- They can only reach the API Traefik ingress (HTTPS)
- They **cannot** directly access databases
- Database credentials aren't on frontend servers

**If an attacker compromises an API worker:**
- They can reach databases via overlay network, but...
- They need valid credentials (not stored in plaintext)
- They **cannot** pivot to other VPN nodes easily
- Swarm secrets manage sensitive config

## Agent Security

HIVE agents use a multi-layer authentication model:

### 1. mTLS (Mutual TLS)

Agents authenticate to the platform using client certificates:

```yaml
# Agent configuration
controlPlane:
  tls:
    enabled: true
    certFile: /etc/hive/agent.crt
    keyFile: /etc/hive/agent.key
    caFile: /etc/hive/ca.crt
```

The platform verifies:
- Certificate is signed by trusted CA
- Certificate is not expired or revoked
- Client certificate fingerprint matches registered agent

### 2. Ed25519 Message Signatures

All job messages from the platform are cryptographically signed:

```
Platform: sign(privateKey, canonicalJson(job)) → signature
Agent: verify(publicKey, canonicalJson(job), signature) → boolean
```

This prevents:
- Message tampering in transit
- Replay attacks (signed messages include timestamps)
- Unauthorized job injection

### 3. Why Both?

| Security Layer | Protects Against |
|----------------|------------------|
| mTLS | Impersonation, man-in-the-middle |
| Ed25519 | Message tampering, replay attacks |
| Combined | Defense in depth |

## User Authentication

### JWT Flow via Keycloak

1. User authenticates with Keycloak
2. Keycloak issues JWT with claims:
   - `sub`: User ID
   - `tenant`: Tenant identifier
   - `org`: Organization ID
   - `roles`: User roles
3. Webapp includes JWT in API requests
4. Platform validates JWT signature via JWKS
5. Tenant isolation filter extracts tenant context

### Tenant Isolation

Every database query is automatically scoped to the current tenant:

```java
// TenantIsolationFilter automatically adds tenant context
// to all MongoDB queries
Query query = new Query()
    .addCriteria(Criteria.where("tenantId").is(currentTenant));
```

## Secrets Management

### Docker Swarm Secrets

Production credentials are stored as Docker Swarm secrets:

```bash
# Create secrets (never in code or env vars)
echo "$MONGO_PASSWORD" | docker secret create mongo_password -
echo "$JWT_SECRET" | docker secret create jwt_secret -
```

Services access secrets as files:
```yaml
services:
  platform:
    secrets:
      - mongo_password
    environment:
      - MONGODB_PASSWORD_FILE=/run/secrets/mongo_password
```

### Credential Rotation

1. Create new secret with version suffix
2. Update service to use new secret
3. Rolling restart replaces old containers
4. Remove old secret

## Firewall Configuration

### Frontend Servers (Public-Facing)

```bash
# Required for web traffic
ALLOW 22/tcp        # SSH (management, restricted IPs)
ALLOW 80/tcp        # HTTP (redirect to HTTPS)
ALLOW 443/tcp       # HTTPS (web traffic)
ALLOW 51820/udp     # WireGuard VPN

# Swarm ports - VPN only
ALLOW from 10.0.0.0/24 to 2377/tcp   # Swarm management
ALLOW from 10.0.0.0/24 to 7946/tcp   # Swarm discovery
ALLOW from 10.0.0.0/24 to 7946/udp   # Swarm discovery
ALLOW from 10.0.0.0/24 to 4789/udp   # Overlay network

DENY all other incoming
```

### Infrastructure Servers (VPN-Only)

```bash
# No public exposure
ALLOW 22/tcp from 10.0.0.0/24   # SSH via VPN only
ALLOW 51820/udp                  # WireGuard
ALLOW from 10.0.0.0/24           # All VPN traffic

DENY all public incoming
```

## High Availability

### No Single Points of Failure

| Component | HA Strategy |
|-----------|-------------|
| VPN | Mesh topology — any node can route traffic |
| Docker Swarm (API) | 3 manager nodes — survives 1 failure |
| Docker Swarm (Frontend) | 3 manager nodes — survives 1 failure |
| MongoDB | 3-node replica set |
| Elasticsearch | 3-node cluster |
| RabbitMQ | Clustered with mirrored queues |
| Keycloak | 2+ replicas with shared PostgreSQL |

### Swarm Manager Quorum

With 3 managers, the cluster maintains quorum if 1 node fails:
- **3 managers**: Need 2 for quorum ✓
- **Raft consensus**: Automatic leader election
- **Self-healing**: Failed containers automatically rescheduled

## Security Checklist

### Pre-Production

- [ ] All services use TLS in transit
- [ ] Database credentials in Docker secrets (not env vars)
- [ ] SSH password authentication disabled
- [ ] Firewall rules configured per above
- [ ] mTLS enabled for agent connections
- [ ] Ed25519 signing enabled for job messages
- [ ] JWT validation configured with correct issuer
- [ ] VPN mesh established between all nodes

### Operational

- [ ] Rotate database passwords quarterly
- [ ] Review Swarm secrets for unused entries
- [ ] Audit SSH access logs
- [ ] Monitor for failed authentication attempts
- [ ] Keep WireGuard, Docker, and dependencies updated

## See Also

- [VPN Setup](vpn-setup.md) - WireGuard mesh configuration
- [Swarm Setup](swarm-setup.md) - Docker Swarm deployment
- [Production Environment](../environments/production.md) - Dual-swarm architecture
- [Agent Connection Flow](../architecture/agent-connection.md) - Authentication details
