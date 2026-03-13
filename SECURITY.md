# Security Policy

## Reporting a Vulnerability

We take security vulnerabilities seriously. If you discover a security issue in the Hive platform, please report it responsibly.

### How to Report

**Do NOT create a public GitHub issue for security vulnerabilities.**

Instead, please email security concerns to: **security@ostec.io**

Include the following information:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Any suggested fixes (optional)

### What to Expect

- **Acknowledgment**: We will acknowledge receipt of your report within 48 hours
- **Assessment**: We will assess the vulnerability and determine its severity within 7 days
- **Resolution**: We aim to release a fix within 30 days for critical issues, 90 days for others
- **Credit**: We will credit you in the release notes (unless you prefer to remain anonymous)

### Scope

This security policy applies to the Hive platform, including:
- The control plane (hive-platform)
- The cluster agent (hive-agent)
- The WebSocket gateway (hive-gateway)
- The dashboard (hive-webapp)
- The shared authentication library (hive-security-starter)

### Out of Scope

- Vulnerabilities in third-party dependencies (report these to the upstream project)
- Social engineering attacks

## Security Best Practices

When deploying Hive:

1. **Use TLS everywhere** - All communication between components should use TLS
2. **Rotate credentials** - Regularly rotate API keys and secrets
3. **Network isolation** - Use VPNs or private networks for inter-component communication
4. **Least privilege** - Grant only necessary permissions to agents and users
5. **Audit logging** - Enable and monitor audit logs for security events

## Security Features

Hive includes several security features:

- Ed25519 cryptographic signatures for agent authentication
- JWT-based authentication with Keycloak integration
- Tenant isolation for multi-tenant deployments
- Audit logging for all operations
- Role-based access control (RBAC)

For more information, see the [documentation](docs/).
