# Hive Configuration Reference

This reference lists the main environment variables for each Hive component. Defaults reflect the values in `application.yml` and config defaults.

## Hive Platform (Control Plane)

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | `9130` | HTTP server port |
| `MONGODB_URI` | `mongodb://localhost:27017/hive-control-plane` | MongoDB connection URI |
| `RABBITMQ_HOST` | `localhost` | RabbitMQ host |
| `RABBITMQ_PORT` | `5672` | RabbitMQ port |
| `RABBITMQ_USERNAME` | `guest` | RabbitMQ username |
| `RABBITMQ_PASSWORD` | `guest` | RabbitMQ password |
| `KEYCLOAK_AUTH_SERVER_URL` | `http://localhost:8080` | Keycloak base URL |
| `KEYCLOAK_REALM` | `platform-dev` | Keycloak realm |
| `REALTIME_GATEWAY_URL` | `http://localhost:9131` | Realtime gateway base URL |
| `REALTIME_GATEWAY_API_KEY` | *(empty)* | API key for gateway publishing |
| `REALTIME_GATEWAY_TENANT_ID` | `hive` | Tenant used for event publishing |
| `PROMETHEUS_BASE_URL` | `http://localhost:9090` | Prometheus base URL |
| `ELASTICSEARCH_BASE_URL` | `http://localhost:9200` | Elasticsearch base URL |
| `ELASTICSEARCH_USERNAME` | *(empty)* | Elasticsearch username |
| `ELASTICSEARCH_PASSWORD` | *(empty)* | Elasticsearch password |
| `NEO4J_URI` | `bolt://localhost:7687` | Neo4j connection URI |
| `NEO4J_USERNAME` | `neo4j` | Neo4j username |
| `NEO4J_PASSWORD` | `password` | Neo4j password |
| `LOGS_STREAM_ENABLED` | `true` | Enable log streaming |
| `LOGS_STREAM_INGEST_LAG_SECONDS` | `10` | Log ingest lag for dedupe |
| `AGENT_WEBSOCKET_PORT` | `8443` | Agent WebSocket port |
| `AGENT_WEBSOCKET_ENABLED` | `true` | Enable agent WebSocket |
| `ED25519_PRIVATE_KEY_PATH` | *(empty)* | Agent message signing private key |
| `ED25519_PUBLIC_KEY_PATH` | *(empty)* | Agent message signing public key |
| `MCP_ENABLED` | `true` | Enable MCP server |
| `MCP_REQUIRE_API_KEY` | `true` | Require API key for MCP |
| `MCP_API_KEY_HEADER` | `X-Hive-Api-Key` | MCP API key header |
| `MCP_RATE_LIMIT_ENABLED` | `true` | Enable MCP rate limiting |

## Hive Gateway (Realtime)

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | `9131` | WebSocket server port |
| `REALTIME_DEV_AUTH_ENABLED` | `false` | Enable dev auth tokens |
| `REALTIME_API_KEYS_INTERNAL` | *(empty)* | Internal publisher API key |
| `REALTIME_API_KEYS_SERVICE` | *(empty)* | Service publisher API key |
| `IDENTITY_ISSUER` | `https://auth.platform.com` | JWT issuer URL |
| `IDENTITY_JWKS_URI` | `https://auth.platform.com/.well-known/jwks.json` | JWKS URL |
| `IDENTITY_AUDIENCE` | `realtime-gateway` | Expected JWT audience |
| `MONGODB_URI` | `mongodb://localhost:27017/hive-gateway` | MongoDB URI |
| `REDIS_HOST` | `localhost` | Redis host |
| `REDIS_PORT` | `6379` | Redis port |
| `WEBSOCKET_ALLOWED_ORIGINS` | *(empty)* | Allowed origins |
| `RATE_LIMIT_PUBLISHER_RPS` | `100` | Publisher rate limit |

## Hive Webapp

| Variable | Default | Description |
|----------|---------|-------------|
| `NEXT_PUBLIC_API_BASE_URL` | `http://localhost:9130` | Control plane API base URL |
| `NEXT_PUBLIC_REALTIME_WS_URL` | `ws://localhost:9131/ws` | Realtime WebSocket URL |
| `NEXT_PUBLIC_REALTIME_GATEWAY_URL` | `http://localhost:9131` | Realtime gateway base URL |
| `NEXT_PUBLIC_DEFAULT_TENANT_ID` | `hive` | Default tenant ID |
| `NEXT_PUBLIC_REALTIME_AUTH_MODE` | `legacy` | Realtime auth mode (`legacy` or `token-in-url`) |
| `NEXT_PUBLIC_BYPASS_AUTH` | `false` | Skip auth (dev only) |
| `NEXT_PUBLIC_DEFAULT_USER_EMAIL` | `demo@example.com` | Dev user email |
| `NEXT_PUBLIC_DEFAULT_USER_PASSWORD` | `Demo123!` | Dev user password |
| `NEXT_PUBLIC_REALTIME_ACCESS_TOKEN` | *(empty)* | Static realtime token (dev only) |
| `KEYCLOAK_URL` | *(required)* | Keycloak base URL (server-side) |
| `KEYCLOAK_REALM` | *(required)* | Keycloak realm (server-side) |
| `KEYCLOAK_CLIENT_ID` | *(required)* | Keycloak client ID (server-side) |
| `KEYCLOAK_CLIENT_SECRET` | *(required)* | Keycloak client secret (server-side) |

## Hive Agent

| Variable | Default | Description |
|----------|---------|-------------|
| `HIVE_CLUSTER_ID` | *(required)* | Cluster ID |
| `HIVE_CONTROL_PLANE_URL` | `wss://localhost:8443/agent/v1/connect` | Control plane WebSocket URL |
| `HIVE_HOSTNAME` | *(auto)* | Agent hostname override |
| `HIVE_AGENT_SERVICE_NAME` | `hive-agent` | Agent service name |
| `DOCKER_HOST` | `unix:///var/run/docker.sock` | Docker socket |
| `HIVE_LOG_LEVEL` | `info` | Log level |
| `HIVE_INTEGRATIONS_ENABLED` | `true` | Enable integration discovery |
| `HIVE_INTEGRATIONS_DISCOVERY_MODE` | `published_ports` | Integration discovery mode |
| `HIVE_INTEGRATIONS_CHECK_INTERVAL` | `30s` | Integration check interval |
| `HIVE_INTEGRATIONS_DIAL_TIMEOUT` | `2s` | Integration dial timeout |
| `HIVE_DNS_ENABLED` | `false` | Enable DNS management |
| `HIVE_DNS_PROVIDER` | `pihole` | DNS provider |
| `HIVE_DNS_MODE` | `published_ports` | DNS mode |
| `HIVE_DNS_DOMAIN` | *(empty)* | DNS domain |
| `HIVE_DNS_DEFAULT_IP` | *(empty)* | Default DNS IP |
| `HIVE_DNS_PIHOLE_BASE_URL` | *(empty)* | Pi-hole base URL |
| `HIVE_DNS_PIHOLE_PASSWORD_SECRET` | *(empty)* | Pi-hole password secret |
| `HIVE_DNS_PIHOLE_PASSWORD_FILE` | *(empty)* | Pi-hole password file |
| `HIVE_DNS_PIHOLE_INSECURE` | `false` | Allow insecure TLS to Pi-hole |
| `HIVE_EVENTS_ENABLED` | `true` | Enable Docker event streaming |
| `HIVE_EVENTS_DEBOUNCE` | `2s` | Docker event debounce |
| `HIVE_METRICS_ENABLED` | `true` | Enable metrics endpoint |
| `HIVE_METRICS_PORT` | `9190` | Metrics port |
| `HIVE_METRICS_PATH` | `/metrics` | Metrics path |
| `HIVE_TLS_CERT_FILE` | *(empty)* | mTLS client cert |
| `HIVE_TLS_KEY_FILE` | *(empty)* | mTLS client key |
| `HIVE_TLS_CA_FILE` | *(empty)* | mTLS CA cert |
| `HIVE_PUBLIC_KEY` | *(empty)* | Ed25519 public key |
| `HIVE_RESTART_POLICY_CONDITION` | `any` | Default restart policy condition |
| `HIVE_RESTART_POLICY_MAX_ATTEMPTS` | `0` | Default restart policy max attempts |
| `HIVE_RESTART_POLICY_DELAY` | `5s` | Default restart policy delay |
