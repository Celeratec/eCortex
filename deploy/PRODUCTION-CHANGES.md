# Production Deployment Changes

This document describes the production deployment configuration required for Traefik v3 compatibility. These changes are now committed to the repository.

## Overview

The production deployment includes critical fixes for:
1. Traefik v3 file provider configuration
2. TLS backend certificate verification for internal connections

---

## Required Files

### 1. Traefik File Provider Configuration

**File:** `deploy/traefik/config.yml`

This file is required because:
- Traefik v3 needs file-based middleware definitions for cross-service middleware usage
- The `security-headers@file` middleware is referenced in docker-compose.yml
- The `insecure` serversTransport configuration skips TLS verification for internal backend connections

---

## Key Configuration Explained

### File Provider (Traefik command)

```yaml
- "--providers.file.directory=/etc/traefik/config"
- "--providers.file.watch=true"
```

Enables Traefik to load middleware and transport configs from files.

### Config File Mount (Traefik volumes)

```yaml
- ./traefik/config.yml:/etc/traefik/config/config.yml:ro
```

Mounts the configuration file into the Traefik container.

### Middleware Reference (MeshCentral labels)

```yaml
- "traefik.http.routers.meshcentral.middlewares=security-headers@file"
```

Uses `@file` suffix to reference middleware defined in the file provider.

### ServersTransport (MeshCentral labels)

```yaml
- "traefik.http.services.meshcentral.loadbalancer.serversTransport=insecure@file"
```

Required because MeshCentral's internal TLS certificate is only valid for `127.0.0.1`, not the Docker container IP (e.g., `172.27.0.x`). Without this, Traefik returns 500 errors.

---

## Why These Changes Are Critical

| Problem | Symptom | Solution |
|---------|---------|----------|
| Missing file provider | `middleware "security-headers@file" does not exist` | Add file provider config |
| TLS certificate mismatch | `x509: certificate is valid for 127.0.0.1, not 172.27.0.4` | Add `insecureSkipVerify` |

---

## Directory Structure

```
deploy/
├── docker-compose.yml          # References @file middlewares
├── traefik/
│   └── config.yml              # Middleware and transport definitions
├── config.json.template
├── init-mongo.js.template
└── env.example
```

---

## Verification

After deployment, verify the configuration is working:

```bash
# Check Traefik logs for errors
docker logs meshcentral-traefik --tail 50 | grep -i error

# Test the website
curl -I https://ecortex.cortalis.com

# Expected: HTTP/2 200 OK with security headers
```

---

## For Future Updates

### DO preserve:
- `deploy/traefik/config.yml`
- File provider commands in Traefik service
- `@file` middleware references
- `serversTransport=insecure@file` configuration

### DO NOT:
- Remove `insecureSkipVerify` (required for internal TLS)
- Change `security-headers@file` to inline labels
- Add CORS middleware to config.yml (not compatible with Traefik v3)

---

## Version Information

| Component | Version |
|-----------|---------|
| Traefik | v3.0 |
| MeshCentral | ghcr.io/ylianst/meshcentral:latest |
| MongoDB | 7 |
| Last Updated | 2026-01-19 |
