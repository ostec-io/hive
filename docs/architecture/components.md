# HIVE Components

## hive-platform (Control Plane)

**Stack:** Java 21, Spring Boot 3.x
**Ports:** 9130 (HTTP API), 8443 (Agent WebSocket)

The central brain of HIVE. Manages cluster registration, dispatches jobs to agents, and maintains the global state of all managed infrastructure.

### Key Features
- Multi-cluster management with real-time health monitoring
- Job dispatch system with Ed25519 cryptographic signatures
- Full Docker Swarm lifecycle management (services, nodes, tasks, secrets)
- Observability integration (Prometheus, Elasticsearch, Neo4j)
- Multi-tenant support with RBAC
- MCP (Model Context Protocol) server for AI integration

### Key Packages
- `io.ostec.hive.control_plane.agent` - Agent communication (WebSocket, sessions)
- `io.ostec.hive.control_plane.service` - Business logic (clusters, deployments)
- `io.ostec.hive.control_plane.mcp` - MCP tools for AI assistants

---

## hive-agent

**Stack:** Go 1.24
**Ports:** 9190 (Prometheus metrics)

Lightweight agent that runs on Docker Swarm manager nodes. Executes jobs from the control plane and reports cluster state.

### Key Features
- WebSocket connection to control plane with automatic reconnection
- Ed25519 signature verification for job authenticity
- Full Docker Swarm API integration via official SDK
- State synchronization (services, nodes, tasks, networks)
- Exec into containers for debugging
- DNS management (CoreDNS, Pi-hole integration)

### Key Packages
- `internal/connection` - WebSocket client with reconnection logic
- `internal/docker` - Docker Swarm client wrapper
- `internal/jobs` - Job executor for all operation types
- `internal/crypto` - Ed25519 signature verification
- `pkg/protocol` - Message types shared with platform

---

## hive-gateway

**Stack:** Java 21, Spring Boot 3.x (Spring MVC)
**Ports:** 9131 (WebSocket)

Real-time event gateway for pushing updates to browsers. Handles WebSocket connections from the dashboard.

### Key Features
- WebSocket server with STOMP support
- Channel-based subscriptions using `/topic/{tenant}/{app}/{channel}`
- Multi-tenant event isolation
- REST API for event publishing
- Redis pub/sub for horizontal scaling

### Key Packages
- `io.ostec.hive.gateway.websocket` - WebSocket handlers
- `io.ostec.hive.gateway.channel` - Channel management
- `io.ostec.hive.gateway.api` - REST endpoints for publishing

---

## hive-webapp

**Stack:** Next.js 15, React 19, TypeScript
**Ports:** 3000 (development), 80/443 (production via Traefik)

Modern dashboard for managing Docker Swarm clusters. Provides real-time visibility and control.

### Key Features
- Cluster overview with health status
- Service management (deploy, scale, rollback)
- Node management (drain, activate, labels)
- Real-time logs and events
- Secret and config management
- Registry integration

### Key Technologies
- React 19 with Server Components
- Tailwind CSS for styling
- WebSocket for real-time updates

---

## hive-security-starter

**Stack:** Java 21, Spring Security 6.x
**Type:** Library (Maven dependency)

Shared authentication library for HIVE Java services. Provides Keycloak JWT integration and multi-tenant isolation.

### Key Features
- JWT validation with JWKS caching
- Keycloak realm/client configuration
- Tenant isolation from JWT claims
- Security context helpers
- Both Servlet and WebFlux support

### Key Classes
- `JwtAuthenticationFilter` - JWT extraction and validation
- `TenantIsolationFilter` - Tenant context propagation
- `SecurityContextHelper` - Static access to current user/tenant
- `AuthUser` - Principal with tenant and org information

---

## Component Interaction Matrix

| From \ To | Platform | Agent | Gateway | Webapp | Keycloak |
|-----------|----------|-------|---------|--------|----------|
| **Platform** | - | WSS :8443 | Internal | - | JWT validation |
| **Agent** | WSS :8443 | - | - | - | - |
| **Gateway** | Internal | - | - | WSS :9131 | - |
| **Webapp** | HTTPS :9130 | - | WSS :9131 | - | OAuth flow |
| **User** | - | - | - | HTTPS :443 | Login |
