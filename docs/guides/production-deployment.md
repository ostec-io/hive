# Production Deployment

## Overview

This guide covers deploying HIVE to a production environment with the dual-swarm architecture: a Frontend Swarm for public-facing services and an API/Infrastructure Swarm for the control plane and databases.

## Operational Notes

- Keep internal services (MongoDB, RabbitMQ, Keycloak, Neo4j, Elasticsearch) reachable only over VPN/overlay networks.
- For Keycloak HA, ensure cluster traffic (JGroups/DB) is allowed between nodes; missing internal connectivity can break session refresh.
- Bake application versions into Docker image tags and set `APP_VERSION` at build time for reproducible releases.

## Prerequisites

- **6+ Linux servers** (Ubuntu 22.04 recommended)
  - 3 for Frontend Swarm
  - 3+ for API/Infrastructure Swarm
- Docker Engine on Linux (Docker Desktop is **not** supported for Swarm/overlay networking)
- **WireGuard VPN** configured between all nodes
- **Domain names** with DNS configured
- **TLS certificates** (Let's Encrypt or similar)

## Step 1: Set Up VPN Mesh

Follow the [VPN Setup Guide](../infrastructure/vpn-setup.md) to create a WireGuard mesh between all servers.

**IP Allocation:**
| Range | Purpose |
|-------|---------|
| 10.0.0.1-3 | API Swarm managers |
| 10.0.0.10-19 | API Swarm workers |
| 10.0.0.20-29 | Frontend Swarm |

## Step 2: Initialize API Swarm

On the first API manager (10.0.0.1):

```bash
docker swarm init --advertise-addr 10.0.0.1

# Get join tokens
docker swarm join-token manager
docker swarm join-token worker
```

Join additional managers and workers:

```bash
# On API managers 2-3
docker swarm join --token <MANAGER_TOKEN> 10.0.0.1:2377

# On API workers
docker swarm join --token <WORKER_TOKEN> 10.0.0.1:2377
```

## Step 3: Initialize Frontend Swarm

On the first frontend manager (10.0.0.20):

```bash
docker swarm init --advertise-addr 10.0.0.20

# Get join tokens
docker swarm join-token manager
```

Join additional frontend managers:

```bash
# On frontend managers 21-22
docker swarm join --token <FRONTEND_MANAGER_TOKEN> 10.0.0.20:2377
```

## Step 4: Deploy Data Services

On the API Swarm, follow the [Swarm Setup Guide](../infrastructure/swarm-setup.md) to deploy:

- MongoDB (3-node replica set)
- Elasticsearch (3-node cluster)
- Neo4j
- RabbitMQ (3-node cluster)
- Keycloak (2 instances + PostgreSQL)

```bash
# Create secrets
docker secret create mongo_root_password -
docker secret create elastic_password -
# ... (see swarm-setup.md for full list)

# Deploy stack
docker stack deploy -c infra-stack.yml infra
```

## Step 5: Deploy HIVE Platform

Create the platform stack file:

```yaml
# hive-platform-stack.yml
version: '3.8'

networks:
  hive-internal:
    external: true
    name: infra_infra-net

services:
  platform:
    image: ghcr.io/ostec-io/hive-platform:<version>
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 30s
    ports:
      - "9130:9130"
      - "8443:8443"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - MONGODB_URI_FILE=/run/secrets/mongodb_uri
      - KEYCLOAK_AUTH_SERVER_URL=http://keycloak:8080
    secrets:
      - mongodb_uri
      - jwt_secret
    networks:
      - hive-internal

  agent:
    image: ghcr.io/ostec-io/hive-agent:<version>
    deploy:
      mode: global
      placement:
        constraints:
          - node.platform.os == linux
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - HIVE_CONTROL_PLANE_URL=wss://platform:8443/agent/v1/connect
      - HIVE_AGENT_TLS_ENABLED=true
      - HIVE_AGENT_CERT_FILE=/run/secrets/agent_cert
      - HIVE_AGENT_KEY_FILE=/run/secrets/agent_key
    secrets:
      - agent_cert
      - agent_key
      - platform_public_key
    networks:
      - hive-internal

secrets:
  mongodb_uri:
    external: true
  jwt_secret:
    external: true
  agent_cert:
    external: true
  agent_key:
    external: true
  platform_public_key:
    external: true
```

Deploy:

```bash
docker stack deploy -c hive-platform-stack.yml hive
```

## Step 6: Deploy Frontend Services

On the **Frontend Swarm**, create the frontend stack:

```yaml
# hive-frontend-stack.yml
version: '3.8'

networks:
  traefik-public:
    driver: overlay
    attachable: true

services:
  traefik:
    image: traefik:v3.0
    deploy:
      placement:
        constraints:
          - node.role == manager
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/traefik:/etc/traefik
      - traefik-certs:/certs
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.swarmMode=true
      - --providers.docker.exposedByDefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.httpchallenge=true
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.letsencrypt.acme.email=admin@example.com
      - --certificatesresolvers.letsencrypt.acme.storage=/certs/acme.json
    networks:
      - traefik-public

  webapp:
    image: ghcr.io/ostec-io/hive-webapp:<version>
    deploy:
      replicas: 2
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.webapp.rule=Host(`app.example.com`)"
        - "traefik.http.routers.webapp.entrypoints=websecure"
        - "traefik.http.routers.webapp.tls.certresolver=letsencrypt"
        - "traefik.http.services.webapp.loadbalancer.server.port=3000"
    environment:
      - NEXT_PUBLIC_API_BASE_URL=https://api.example.com
      - NEXT_PUBLIC_REALTIME_WS_URL=wss://ws.example.com/ws
      - KEYCLOAK_URL=https://auth.example.com
      - KEYCLOAK_REALM=hive
      - KEYCLOAK_CLIENT_ID=hive-webapp
      - KEYCLOAK_CLIENT_SECRET=${KEYCLOAK_CLIENT_SECRET}
    networks:
      - traefik-public

  gateway:
    image: ghcr.io/ostec-io/hive-gateway:<version>
    deploy:
      replicas: 2
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.gateway.rule=Host(`ws.example.com`)"
        - "traefik.http.routers.gateway.entrypoints=websecure"
        - "traefik.http.routers.gateway.tls.certresolver=letsencrypt"
        - "traefik.http.services.gateway.loadbalancer.server.port=9131"
    networks:
      - traefik-public

volumes:
  traefik-certs:
```

Deploy:

```bash
docker stack deploy -c hive-frontend-stack.yml frontend
```

## Step 7: Configure DNS

Set up DNS records pointing to your Frontend Swarm's public IP:

| Record | Type | Value |
|--------|------|-------|
| `app.example.com` | A | `<frontend_public_ip>` |
| `ws.example.com` | A | `<frontend_public_ip>` |
| `api.example.com` | A | `<frontend_public_ip>` |
| `auth.example.com` | A | `<frontend_public_ip>` |

## Step 8: Enable mTLS for Agents

Generate certificates for agent authentication:

```bash
# Generate CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/CN=HIVE CA"

# Generate agent certificate
openssl genrsa -out agent.key 2048
openssl req -new -key agent.key -out agent.csr -subj "/CN=hive-agent"
openssl x509 -req -days 365 -in agent.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out agent.crt

# Create Docker secrets
docker secret create agent_cert agent.crt
docker secret create agent_key agent.key
docker secret create ca_cert ca.crt
```

## Step 9: Generate Ed25519 Keys

For job signing:

```bash
# Generate Ed25519 keypair
openssl genpkey -algorithm Ed25519 -out ed25519-private.pem
openssl pkey -in ed25519-private.pem -pubout -out ed25519-public.pem

# Create secrets
docker secret create ed25519_private ed25519-private.pem
docker secret create ed25519_public ed25519-public.pem
```

## Step 10: Verify Deployment

### Check Services

```bash
# API Swarm
docker service ls

# Frontend Swarm (run on frontend manager)
docker service ls
```

### Test Endpoints

```bash
# Platform API
curl https://api.example.com/actuator/health

# Webapp
curl -I https://app.example.com

# Keycloak
curl https://auth.example.com/health/ready
```

### Check Agent Connection

```bash
docker service logs hive_agent
```

## MCP Access (Optional)

MCP clients call the **control plane API** over HTTPS.
In a dual‑swarm setup this typically routes:

`MCP client → Frontend Traefik → API Traefik → hive-platform`

If you keep the API VPN‑only, MCP callers must be on the VPN.
If you expose it publicly, ensure you require API keys/JWTs and rate‑limit appropriately.

## Security Checklist

- [ ] WireGuard VPN mesh established
- [ ] Firewall rules configured (see [security.md](../infrastructure/security.md))
- [ ] TLS certificates installed
- [ ] mTLS enabled for agents
- [ ] Ed25519 signing enabled
- [ ] Docker secrets used for all credentials
- [ ] SSH password auth disabled
- [ ] Database ports not exposed to VPN

## Monitoring Setup

Deploy Prometheus and Grafana for observability:

```bash
# Add to API Swarm
docker service create \
  --name prometheus \
  --network infra_infra-net \
  -p 9090:9090 \
  prom/prometheus

docker service create \
  --name grafana \
  --network infra_infra-net \
  -p 3001:3000 \
  grafana/grafana
```

## Backup Strategy

Schedule regular backups:

```bash
# MongoDB
mongodump --uri="mongodb://..." --out=/backup/mongodb/$(date +%Y%m%d)

# PostgreSQL (Keycloak)
pg_dump -U keycloak keycloak > /backup/keycloak/$(date +%Y%m%d).sql

# Neo4j
neo4j-admin database dump neo4j --to-path=/backup/neo4j/$(date +%Y%m%d)
```

## Next Steps

- [Adding Clusters](adding-clusters.md) - Scale with more clusters
- [Security Architecture](../infrastructure/security.md) - Zero-trust details
- [Networking Reference](../infrastructure/networking.md) - Port matrix
