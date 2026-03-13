# Getting Started

## Overview

This guide walks you through setting up HIVE locally for development. By the end, you'll have all components running on a single Docker Swarm cluster.

## Prerequisites

- **Docker Engine on Linux** with Swarm mode enabled  
  (Docker Desktop is **not** supported for Swarm/overlay networking. Use a Linux host or Linux VM.)
- **8GB RAM** minimum (16GB recommended)
- **Ports available**: 3000, 9131, 8080, 8443, 9130, 27017

Why not Docker Desktop? It commonly breaks Swarm overlay DNS, advertises the wrong IP behind NAT,
and causes agents to report stale or incorrect state.  
See: [Linux VM Setup Guide](linux-vm.md)

## Quick Start with Docker Compose

The fastest way to get started:

```bash
# Clone the repos (or create docker-compose.yml)
mkdir hive && cd hive

# Create docker-compose.yml
cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    ports:
      - "8080:8080"
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    command: start-dev

  platform:
    image: ghcr.io/ostec-io/hive-platform:latest
    ports:
      - "9130:9130"
      - "8443:8443"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/hive
      - KEYCLOAK_AUTH_SERVER_URL=http://keycloak:8080
    depends_on:
      - mongodb
      - keycloak

  agent:
    image: ghcr.io/ostec-io/hive-agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - HIVE_CONTROL_PLANE_URL=ws://platform:8443/agent/v1/connect

  gateway:
    image: ghcr.io/ostec-io/hive-gateway:latest
    ports:
      - "9131:9131"

  webapp:
    image: ghcr.io/ostec-io/hive-webapp:latest
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_BASE_URL=http://localhost:9130
      - NEXT_PUBLIC_REALTIME_WS_URL=ws://localhost:9131/ws
      - KEYCLOAK_URL=http://localhost:8080
      - KEYCLOAK_REALM=hive
      - KEYCLOAK_CLIENT_ID=hive-webapp
      - KEYCLOAK_CLIENT_SECRET=dev-secret

volumes:
  mongodb_data:
EOF

# Start everything
docker-compose up -d

# Watch logs
docker-compose logs -f
```

## Access the Dashboard

Once all services are running:

1. Open http://localhost:3000 in your browser
2. Log in with Keycloak (admin/admin for dev)
3. The agent should automatically connect and register the local swarm

## Alternative: Docker Swarm Mode

For a setup closer to production:

### 1. Initialize Swarm

```bash
docker swarm init
```

### 2. Create Overlay Network

```bash
docker network create --driver overlay --attachable hive-network
```

### 3. Start Data Services

```bash
# MongoDB
docker service create \
  --name mongodb \
  --network hive-network \
  -p 27017:27017 \
  mongo:7

# Keycloak
docker service create \
  --name keycloak \
  --network hive-network \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:24.0 start-dev
```

### 4. Start HIVE Services

```bash
# Platform
docker service create \
  --name hive-platform \
  --network hive-network \
  -p 9130:9130 \
  -p 8443:8443 \
  -e MONGODB_URI=mongodb://mongodb:27017/hive \
  -e KEYCLOAK_AUTH_SERVER_URL=http://keycloak:8080 \
  ghcr.io/ostec-io/hive-platform:latest

# Agent
docker service create \
  --name hive-agent \
  --network hive-network \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
  -e HIVE_CONTROL_PLANE_URL=ws://hive-platform:8443/agent/v1/connect \
  ghcr.io/ostec-io/hive-agent:latest

# Gateway
docker service create \
  --name hive-gateway \
  --network hive-network \
  -p 9131:9131 \
  ghcr.io/ostec-io/hive-gateway:latest

# Webapp
docker service create \
  --name hive-webapp \
  --network hive-network \
  -p 3000:3000 \
  -e NEXT_PUBLIC_API_BASE_URL=http://localhost:9130 \
  -e NEXT_PUBLIC_REALTIME_WS_URL=ws://localhost:9131/ws \
  ghcr.io/ostec-io/hive-webapp:latest
```

## Verify Installation

### Check Services

```bash
# Docker Compose
docker-compose ps

# Docker Swarm
docker service ls
```

### Check Platform Health

```bash
curl http://localhost:9130/actuator/health
```

Expected response:
```json
{"status": "UP"}
```

### Check Agent Connection

```bash
docker logs $(docker ps -q -f name=hive-agent)
```

Look for:
```
Connected to control plane
Sending AGENT_HELLO
Received HIVE_HELLO, assigned agentId: agent-xyz789
```

## Configure Keycloak

For a complete setup, configure Keycloak with a HIVE realm:

1. Open http://localhost:8080
2. Log in with admin/admin
3. Create a new realm called "hive"
4. Create a client for the webapp
5. Configure redirect URIs for http://localhost:3000

## Development Workflow

### Building from Source

If you want to modify components:

```bash
# Clone repos
git clone https://github.com/ostec-io/hive-platform.git
git clone https://github.com/ostec-io/hive-agent.git
git clone https://github.com/ostec-io/hive-gateway.git
git clone https://github.com/ostec-io/hive-webapp.git

# Build platform (Java)
cd hive-platform
./mvnw clean package -DskipTests
docker build -t hive-platform:dev .

# Build agent (Go)
cd ../hive-agent
go build -o hive-agent ./cmd/agent
docker build -t hive-agent:dev .

# Build webapp (Node.js)
cd ../hive-webapp
npm install
npm run build
docker build -t hive-webapp:dev .
```

### Hot Reloading

For faster development iteration:

```bash
# Webapp with hot reload
cd hive-webapp
npm run dev

# Platform with Spring Boot DevTools
cd hive-platform
./mvnw spring-boot:run

# Agent
cd hive-agent
go run ./cmd/agent
```

## Troubleshooting

### Agent Not Connecting

1. Check platform is running:
   ```bash
   docker service logs hive-platform
   ```

2. Verify network connectivity:
   ```bash
   docker exec -it $(docker ps -q -f name=hive-agent) ping hive-platform
   ```

3. Check WebSocket port:
   ```bash
   wscat -c ws://localhost:8443/agent/v1/connect
   ```
   If you’ve configured TLS/mTLS on the control plane, use `wss://` instead.

### Webapp Shows "API Unreachable"

1. Verify platform health:
   ```bash
   curl http://localhost:9130/actuator/health
   ```

2. Check CORS settings if running webapp from source

3. Ensure environment variables point to correct URLs

### MongoDB Connection Failed

1. Check MongoDB is running:
   ```bash
   docker logs $(docker ps -q -f name=mongodb)
   ```

2. Test connection:
   ```bash
   docker exec -it $(docker ps -q -f name=mongodb) mongosh
   ```

## Next Steps

- [Development Environment](../environments/development.md) - Detailed dev setup
- [Production Deployment](production-deployment.md) - Deploy to production
- [Architecture Overview](../architecture/overview.md) - Understand the system
