# Docker Swarm Setup

## Overview

HIVE uses Docker Swarm for container orchestration. This guide covers deploying the data services stack (MongoDB, Elasticsearch, Neo4j, RabbitMQ, Keycloak) on a 3-node cluster.

## Why Docker Swarm?

| Feature | Docker Compose | Docker Swarm |
|---------|----------------|--------------|
| Deployment | Per-node | Single `docker stack deploy` |
| Service Discovery | Manual | Automatic DNS |
| Load Balancing | External | Built-in routing mesh |
| Secrets | `.env` files | Encrypted secrets |
| Failover | Manual | Automatic rescheduling |
| Scaling | Manual | `docker service scale` |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Docker Swarm Cluster                                │
│                    Overlay Network: infra-net (encrypted)                   │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│   Node 1 (Manager)  │   Node 2 (Worker)   │      Node 3 (Worker)            │
│     10.0.0.1        │     10.0.0.2        │        10.0.0.3                 │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ mongo-1 (PRIMARY)   │ mongo-2 (SECONDARY) │ mongo-3 (SECONDARY)             │
│ elasticsearch-1     │ elasticsearch-2     │ elasticsearch-3                 │
│ neo4j               │                     │                                 │
│ rabbitmq-1          │ rabbitmq-2          │ rabbitmq-3                      │
│ keycloak-1          │ keycloak-2          │                                 │
│ keycloak-db         │                     │                                 │
│ traefik             │                     │                                 │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

## Step 1: Initialize Swarm

### On Node 1 (Manager)

```bash
# Initialize swarm using WireGuard VPN IP
docker swarm init --advertise-addr 10.0.0.1

# Save the join token
docker swarm join-token worker
```

### On Node 2 & Node 3 (Workers)

```bash
docker swarm join --token <SWARM_TOKEN> 10.0.0.1:2377
```

### Verify Cluster

```bash
docker node ls
```

Expected output:
```
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS
abc123 *                      node1      Ready     Active         Leader
def456                        node2      Ready     Active
ghi789                        node3      Ready     Active
```

## Step 2: Label Nodes for Placement

```bash
# Run on manager node
docker node update --label-add mongo.replica=1 node1
docker node update --label-add mongo.replica=2 node2
docker node update --label-add mongo.replica=3 node3

docker node update --label-add elasticsearch.node=1 node1
docker node update --label-add elasticsearch.node=2 node2
docker node update --label-add elasticsearch.node=3 node3

docker node update --label-add rabbitmq.node=1 node1
docker node update --label-add rabbitmq.node=2 node2
docker node update --label-add rabbitmq.node=3 node3

docker node update --label-add neo4j=true node1
docker node update --label-add keycloak=true node1
docker node update --label-add keycloak=true node2
docker node update --label-add traefik=true node1
```

## Step 3: Create Docker Secrets

```bash
# Generate secure passwords
MONGO_ROOT_PASS=$(openssl rand -base64 24)
MONGO_KEYFILE=$(openssl rand -base64 756)
ELASTIC_PASS=$(openssl rand -base64 24)
NEO4J_PASS=$(openssl rand -base64 24)
RABBITMQ_PASS=$(openssl rand -base64 24)
ERLANG_COOKIE=$(openssl rand -hex 32)
KC_DB_PASS=$(openssl rand -base64 24)
KC_ADMIN_PASS=$(openssl rand -base64 24)

# Create secrets
echo "$MONGO_ROOT_PASS" | docker secret create mongo_root_password -
echo "$MONGO_KEYFILE" | docker secret create mongo_keyfile -
echo "$ELASTIC_PASS" | docker secret create elastic_password -
echo "$NEO4J_PASS" | docker secret create neo4j_password -
echo "$RABBITMQ_PASS" | docker secret create rabbitmq_password -
echo "$ERLANG_COOKIE" | docker secret create erlang_cookie -
echo "$KC_DB_PASS" | docker secret create keycloak_db_password -
echo "$KC_ADMIN_PASS" | docker secret create keycloak_admin_password -

# Save passwords securely (protect this file!)
cat << EOF > ~/cluster-secrets.txt
MONGO_ROOT_PASS=$MONGO_ROOT_PASS
ELASTIC_PASS=$ELASTIC_PASS
NEO4J_PASS=$NEO4J_PASS
RABBITMQ_PASS=$RABBITMQ_PASS
KC_DB_PASS=$KC_DB_PASS
KC_ADMIN_PASS=$KC_ADMIN_PASS
EOF
chmod 600 ~/cluster-secrets.txt

# Verify secrets
docker secret ls
```

## Step 4: Create Data Directories

Run on **all nodes**:

```bash
sudo mkdir -p /data/{mongodb,elasticsearch,neo4j,rabbitmq,keycloak,traefik}
sudo chown -R 1000:1000 /data/elasticsearch
sudo chown -R 1000:1000 /data/neo4j
sudo chmod 700 /data/mongodb

# Elasticsearch requirement
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 5: Deploy the Stack

Create the stack file and deploy:

```bash
# Deploy the infrastructure stack
docker stack deploy -c infra-stack.yml infra
```

### Monitor Deployment

```bash
# Watch services come up
watch docker service ls

# Check specific service logs
docker service logs infra_mongo-1 -f

# View task status
docker stack ps infra
```

## Step 6: Post-Deployment Configuration

### Initialize MongoDB Replica Set

The `mongo-init` service runs automatically, but verify:

```bash
# Check replica set status
docker exec $(docker ps -q -f name=infra_mongo-1) mongosh \
  -u admin -p "$(cat ~/cluster-secrets.txt | grep MONGO_ROOT | cut -d= -f2)" \
  --eval 'rs.status()'
```

### Join RabbitMQ Cluster

```bash
# Get container IDs
RABBIT1=$(docker ps -q -f name=infra_rabbitmq-1)
RABBIT2=$(docker ps -q -f name=infra_rabbitmq-2)
RABBIT3=$(docker ps -q -f name=infra_rabbitmq-3)

# Join node 2
docker exec $RABBIT2 rabbitmqctl stop_app
docker exec $RABBIT2 rabbitmqctl reset
docker exec $RABBIT2 rabbitmqctl join_cluster rabbit@rabbitmq-1
docker exec $RABBIT2 rabbitmqctl start_app

# Join node 3
docker exec $RABBIT3 rabbitmqctl stop_app
docker exec $RABBIT3 rabbitmqctl reset
docker exec $RABBIT3 rabbitmqctl join_cluster rabbit@rabbitmq-1
docker exec $RABBIT3 rabbitmqctl start_app

# Set HA policy
docker exec $RABBIT1 rabbitmqctl set_policy ha-all ".*" \
  '{"ha-mode":"all","ha-sync-mode":"automatic"}' --apply-to queues

# Verify
docker exec $RABBIT1 rabbitmqctl cluster_status
```

### Verify Elasticsearch Cluster

```bash
ELASTIC_PASS=$(grep ELASTIC ~/cluster-secrets.txt | cut -d= -f2)
curl -u elastic:$ELASTIC_PASS http://10.0.0.1:9200/_cluster/health?pretty
```

## Service Endpoints

### Internal (via Swarm DNS)

| Service | Internal URL |
|---------|--------------|
| MongoDB | `mongodb://admin:pass@mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0` |
| Elasticsearch | `http://elasticsearch-1:9200` |
| Neo4j Bolt | `bolt://neo4j:7687` |
| Neo4j HTTP | `http://neo4j:7474` |
| RabbitMQ | `amqp://admin:pass@rabbitmq-1:5672` |
| Keycloak | `http://keycloak:8080` |

### External (via VPN IPs)

| Service | Node 1 | Node 2 | Node 3 |
|---------|--------|--------|--------|
| MongoDB | `10.0.0.1:27017` | `10.0.0.2:27017` | `10.0.0.3:27017` |
| Elasticsearch | `10.0.0.1:9200` | `10.0.0.2:9200` | `10.0.0.3:9200` |
| Neo4j | `10.0.0.1:7474/7687` | - | - |
| RabbitMQ | `10.0.0.1:5672/15672` | `10.0.0.2:5672/15672` | `10.0.0.3:5672/15672` |
| Keycloak | `10.0.0.1:8080` | | |

## HIVE Platform Configuration

Configure the HIVE platform to connect to the data services:

```yaml
# application-prod.yml
spring:
  data:
    mongodb:
      uri: mongodb://admin:${MONGO_PASSWORD}@mongo-1:27017,mongo-2:27017,mongo-3:27017/hive?replicaSet=rs0&authSource=admin

  elasticsearch:
    uris: http://elasticsearch-1:9200,http://elasticsearch-2:9200,http://elasticsearch-3:9200
    username: elastic
    password: ${ELASTIC_PASSWORD}

  neo4j:
    uri: bolt://neo4j:7687
    authentication:
      username: neo4j
      password: ${NEO4J_PASSWORD}

  rabbitmq:
    addresses: rabbitmq-1:5672,rabbitmq-2:5672,rabbitmq-3:5672
    username: admin
    password: ${RABBITMQ_PASSWORD}

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8080/realms/hive
```

## Management Commands

```bash
# View all services
docker service ls

# Scale a service
docker service scale infra_keycloak=3

# Update a service image
docker service update --image mongo:7.1 infra_mongo-1

# Rolling update all MongoDB nodes
for i in 1 2 3; do
  docker service update --image mongo:7.1 infra_mongo-$i
  sleep 30
done

# View service logs
docker service logs infra_elasticsearch-1 -f --tail 100

# Remove stack
docker stack rm infra

# Force rebalance services
docker node update --availability drain node2
docker node update --availability active node2
```

## Health Check Script

```bash
#!/bin/bash
# cluster-health.sh

source ~/cluster-secrets.txt

echo "=== Docker Swarm Cluster Health ==="

echo "Swarm Nodes:"
docker node ls

echo "Services:"
docker service ls

echo "MongoDB Replica Set:"
docker exec $(docker ps -q -f name=infra_mongo-1) mongosh \
  -u admin -p "$MONGO_ROOT_PASS" --quiet \
  --eval 'rs.status().members.forEach(m => print(m.name + ": " + m.stateStr))'

echo "Elasticsearch Cluster:"
curl -s -u elastic:$ELASTIC_PASS http://10.0.0.1:9200/_cluster/health | \
  jq '{status, number_of_nodes, active_shards}'

echo "RabbitMQ Cluster:"
docker exec $(docker ps -q -f name=infra_rabbitmq-1) rabbitmqctl cluster_status 2>/dev/null | \
  grep -A5 "Running Nodes"

echo "Keycloak:"
curl -s http://10.0.0.1:8080/health/ready | jq

echo "Neo4j:"
curl -s http://10.0.0.1:7474 && echo " Running"
```

## Troubleshooting

### Service Not Starting

```bash
# Check service status
docker service ps infra_<service> --no-trunc

# View logs
docker service logs infra_<service> -f

# Check node constraints
docker node inspect <node> --format '{{ .Spec.Labels }}'
```

### Network Issues

```bash
# List networks
docker network ls

# Inspect overlay network
docker network inspect infra_infra-net

# Check service connectivity
docker exec <container> ping <service-name>
```

### Resource Constraints

```bash
# Check node resources
docker node inspect <node> --format '{{ .Description.Resources }}'

# View service resource usage
docker stats
```

## See Also

- [VPN Setup](vpn-setup.md) - WireGuard mesh configuration
- [Security Architecture](security.md) - Zero-trust model
- [Production Environment](../environments/production.md) - Dual-swarm architecture
