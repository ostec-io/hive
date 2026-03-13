# Hive

**Docker Swarm management platform with real-time monitoring, drift detection, and AI-powered operations.**

Hive provides a modern control plane for managing Docker Swarm clusters at scale, with features typically found only in enterprise orchestration platforms.

## Features

- **Multi-Cluster Management** - Manage multiple Docker Swarm clusters from a single control plane
- **Real-Time Monitoring** - Live service status, metrics, and log streaming via WebSocket
- **Drift Detection** - Automatic detection when running services deviate from desired state
- **Secrets Management** - Secure storage and rotation of secrets across clusters
- **MCP Integration** - AI assistant integration via Model Context Protocol for natural language operations
- **Modern Dashboard** - Clean, responsive web UI built with Next.js

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Hive Platform                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Webapp    │  │   Gateway   │  │        Platform         │  │
│  │  (Next.js)  │──│ (WebSocket) │──│    (Control Plane)      │  │
│  └─────────────┘  └─────────────┘  └───────────┬─────────────┘  │
│                                                 │                │
└─────────────────────────────────────────────────┼────────────────┘
                                                  │
              ┌───────────────────────────────────┼───────────────┐
              │                                   │               │
     ┌────────▼────────┐              ┌───────────▼───────────┐   │
     │   Hive Agent    │              │      Hive Agent       │   │
     │  (Cluster A)    │              │     (Cluster B)       │   │
     └────────┬────────┘              └───────────┬───────────┘   │
              │                                   │               │
     ┌────────▼────────┐              ┌───────────▼───────────┐   │
     │  Docker Swarm   │              │    Docker Swarm       │   │
     │    Cluster      │              │      Cluster          │   │
     └─────────────────┘              └───────────────────────┘   │
```

## Components

| Component | Description | Tech Stack |
|-----------|-------------|------------|
| Platform | Control plane API and business logic | Java 21, Spring Boot |
| Agent | Lightweight agent deployed per cluster | Go |
| Gateway | Real-time WebSocket gateway | Java 21, Spring Boot |
| Webapp | Dashboard web application | Next.js 15, TypeScript |
| Security Starter | Shared authentication library | Java 21, Spring Boot |

## Documentation

This repository contains the architecture and technical documentation for the Hive platform.

- [Architecture Overview](./docs/architecture/overview.md)
- [Component Details](./docs/architecture/components.md)
- [Agent Connection Flow](./docs/architecture/agent-connection.md)
- [Data Flow](./docs/architecture/data-flow.md)
- [Configuration Reference](./docs/configuration-reference.md)
- [Production Deployment](./docs/guides/production-deployment.md)

## Security

For security concerns, please see our [Security Policy](./SECURITY.md).

## License

The documentation in this repository is licensed under the Creative Commons Attribution 4.0 International License - see the [LICENSE](./LICENSE) file for details.

---

Built by [OSTEC](https://ostec.io)
