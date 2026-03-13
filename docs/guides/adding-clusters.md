# Adding Managed Clusters

## Overview

HIVE can manage multiple Docker Swarm clusters from a single control plane. This guide covers adding new clusters to an existing HIVE deployment.

## Prerequisites

- HIVE platform already deployed and running
- New Docker Swarm cluster initialized
- Network connectivity to the new cluster
- mTLS certificates for agent authentication

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HIVE Control Plane                          │
│                                                                     │
│  hive-platform (:9130, :8443)                                      │
│       │                                                             │
│       ├──── WSS ──── hive-agent (Cluster A - Production)           │
│       │                                                             │
│       ├──── WSS ──── hive-agent (Cluster B - Staging)              │
│       │                                                             │
│       └──── WSS ──── hive-agent (Cluster C - Development)          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Step 1: Prepare the New Cluster

### Initialize Swarm (if not already done)

```bash
# On the new cluster's first manager
docker swarm init --advertise-addr <manager_ip>

# Join additional managers
docker swarm join --token <MANAGER_TOKEN> <manager_ip>:2377

# Join workers
docker swarm join --token <WORKER_TOKEN> <manager_ip>:2377
```

### Verify Cluster

```bash
docker node ls
```

## Step 2: Generate Agent Certificates

Each cluster needs unique certificates for mTLS authentication.

### Using the HIVE CA

```bash
# Generate key pair for the new cluster
openssl genrsa -out cluster-b-agent.key 2048

# Create CSR with unique CN
openssl req -new \
  -key cluster-b-agent.key \
  -out cluster-b-agent.csr \
  -subj "/CN=hive-agent-cluster-b/O=HIVE"

# Sign with CA (from your CA server)
openssl x509 -req \
  -days 365 \
  -in cluster-b-agent.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out cluster-b-agent.crt
```

### Create Secrets on New Cluster

```bash
# Transfer files to new cluster, then:
docker secret create agent_cert cluster-b-agent.crt
docker secret create agent_key cluster-b-agent.key
docker secret create ca_cert ca.crt
```

## Step 3: Get Platform Public Key

The agent needs the platform's Ed25519 public key for signature verification.

```bash
# From your secrets store or platform config
cat ed25519-public.pem
# MCowBQYDK2VwAyEA...
```

Create the secret on the new cluster:

```bash
echo "MCowBQYDK2VwAyEA..." | docker secret create platform_public_key -
```

## Step 4: Deploy Agent

Create the agent service on the new cluster:

```yaml
# hive-agent-stack.yml
version: '3.8'

services:
  agent:
    image: ghcr.io/ostec-io/hive-agent:latest
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Point to your HIVE platform
      - HIVE_CONTROL_PLANE_URL=wss://api.example.com:8443/agent/v1/connect

      # Enable TLS
      - HIVE_AGENT_TLS_ENABLED=true
      - HIVE_AGENT_CERT_FILE=/run/secrets/agent_cert
      - HIVE_AGENT_KEY_FILE=/run/secrets/agent_key
      - HIVE_AGENT_CA_FILE=/run/secrets/ca_cert

      # Ed25519 verification
      - HIVE_AGENT_PUBLIC_KEY_FILE=/run/secrets/platform_public_key

      # Cluster identification
      - HIVE_CLUSTER_ID=cluster-b-staging
    secrets:
      - agent_cert
      - agent_key
      - ca_cert
      - platform_public_key

secrets:
  agent_cert:
    external: true
  agent_key:
    external: true
  ca_cert:
    external: true
  platform_public_key:
    external: true
```

Deploy:

```bash
docker stack deploy -c hive-agent-stack.yml hive
```

## Step 5: Verify Connection

### Check Agent Logs

```bash
docker service logs hive_agent -f
```

Look for:
```
Connecting to wss://api.example.com:8443/agent/v1/connect
TLS handshake complete
Sending AGENT_HELLO (clusterId: cluster-b-staging)
Received HIVE_HELLO, assigned agentId: agent-xyz789
State sync requested, gathering cluster state...
State sync complete
```

### Check Platform Logs

```bash
# On the platform swarm
docker service logs hive_platform | grep cluster-b
```

Look for:
```
Agent connected: cluster-b-staging (hostname: manager-1)
Agent registered as leader for cluster-b-staging
```

### Verify in Dashboard

1. Open the HIVE dashboard
2. Navigate to Clusters
3. The new cluster should appear with status "Connected"

## Network Requirements

### From New Cluster to Platform

The agent needs outbound connectivity to the platform:

| Destination | Port | Protocol |
|-------------|------|----------|
| api.example.com | 8443 | WSS (HTTPS) |

### Firewall Configuration

On the new cluster's managers:

```bash
# Allow outbound HTTPS
ufw allow out 443/tcp

# If platform is on VPN, ensure VPN connectivity
# (see VPN setup guide for adding new nodes)
```

## Multi-Cluster Considerations

### Cluster Naming

Use consistent, descriptive cluster IDs:

| Cluster ID | Purpose |
|------------|---------|
| `prod-us-east` | US East production |
| `prod-eu-west` | EU West production |
| `staging` | Pre-production testing |
| `dev` | Development/testing |

### Agent Configuration Per Environment

Production clusters should have stricter security:

```yaml
# Production agent config
environment:
  - HIVE_AGENT_TLS_ENABLED=true
  - HIVE_AGENT_VERIFY_SIGNATURES=true
  - HIVE_CLUSTER_ID=prod-us-east
```

Development clusters can be more relaxed:

```yaml
# Development agent config
environment:
  - HIVE_AGENT_TLS_ENABLED=false  # OK for dev
  - HIVE_AGENT_VERIFY_SIGNATURES=false
  - HIVE_CLUSTER_ID=dev
```

## Troubleshooting

### Agent Can't Connect

1. **Check network connectivity:**
   ```bash
   curl -I https://api.example.com:8443
   ```

2. **Check certificate validity:**
   ```bash
   openssl x509 -in cluster-b-agent.crt -noout -dates
   ```

3. **Verify CA trust:**
   ```bash
   openssl verify -CAfile ca.crt cluster-b-agent.crt
   ```

### Agent Connects but Disconnects

1. **Check heartbeat settings:**
   - Default: 10 seconds
   - Platform may disconnect agents that don't heartbeat

2. **Check for certificate fingerprint mismatch:**
   - Platform logs will show rejection reason

### State Sync Fails

1. **Check Docker socket permissions:**
   ```bash
   ls -la /var/run/docker.sock
   ```

2. **Verify manager node:**
   ```bash
   docker node inspect self --format '{{ .ManagerStatus }}'
   ```

## Removing a Cluster

To remove a cluster from HIVE management:

1. Stop the agent on the cluster:
   ```bash
   docker stack rm hive
   ```

2. The platform will detect the disconnect and mark the cluster as offline

3. Optionally, delete the cluster from the HIVE dashboard

## See Also

- [Agent Connection Flow](../architecture/agent-connection.md) - Auth details
- [Security Architecture](../infrastructure/security.md) - mTLS and Ed25519
- [VPN Setup](../infrastructure/vpn-setup.md) - Adding nodes to VPN mesh
