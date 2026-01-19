# Production Deployment Changes

This document describes the changes made to the production deployment that differ from the upstream Git repository. These changes are **required** for the deployment to work correctly with Traefik v3 and must be preserved in future updates.

## Overview

The production deployment includes critical fixes for Traefik v3 compatibility and TLS backend certificate verification that are not present in the Git repository.

## Required Files

### 1. Traefik File Provider Configuration

**File:** `deploy/traefik/config.yml`  
**Status:** MISSING from Git repository - MUST BE ADDED

This file is required because:
- Traefik v3 needs file-based middleware definitions for cross-service middleware usage
- The `security-headers@file` middleware is referenced in docker-compose.yml but defined here
- The `insecure` serversTransport configuration is needed to skip TLS verification for internal backend connections

**Full file content:**

```yaml
# Traefik Static Configuration - File Provider
# This file defines middlewares that are used across the deployment

http:
  middlewares:
    # Security headers middleware
    security-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        forceSTSHeader: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "strict-origin-when-cross-origin"
        customFrameOptionsValue: "DENY"
        customBrowserXssValue: "1; mode=block"
        permissionsPolicy: "geolocation=(), microphone=(), camera=()"

    # Rate limiting middleware
    ratelimit:
      rateLimit:
        average: 100
        period: 60s
        burst: 50

    # Compression middleware
    compression:
      compress:
        minResponseBodyBytes: 1024


  serversTransports:
    insecure:
      insecureSkipVerify: true
```

**Location:** Create this file at `deploy/traefik/config.yml` in the Git repository.

---

## Required Changes to Existing Files

### 2. docker-compose.yml Modifications

The following changes must be applied to `deploy/docker-compose.yml`:

#### A. Add Traefik File Provider Support

**Location:** In the `traefik` service, under `command:` section (after line 22)

**Add these two lines:**
```yaml
      - "--providers.file.directory=/etc/traefik/config"
      - "--providers.file.watch=true"
```

**Full context (lines 15-28 should look like):**
```yaml
    command:
      # API and Dashboard
      - "--api.dashboard=true"
      - "--api.insecure=false"
      # Providers
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=meshcentral-network"
      - "--providers.file.directory=/etc/traefik/config"
      - "--providers.file.watch=true"
      # Entrypoints
      - "--entrypoints.web.address=:80"
```

#### B. Mount Traefik Config File

**Location:** In the `traefik` service, under `volumes:` section (after line 43)

**Add this line:**
```yaml
      - ./traefik/config.yml:/etc/traefik/config/config.yml:ro
```

**Full context (lines 40-44 should look like):**
```yaml
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-letsencrypt:/letsencrypt
      - traefik-logs:/var/log/traefik
      - ./traefik/config.yml:/etc/traefik/config/config.yml:ro
```

#### C. Update MeshCentral Service Middleware Reference

**Location:** In the `meshcentral` service, line 133

**Change FROM:**
```yaml
      - "traefik.http.routers.meshcentral.middlewares=security-headers,ratelimit"
```

**Change TO:**
```yaml
      - "traefik.http.routers.meshcentral.middlewares=security-headers@file"
```

**Reason:** The middleware is now defined in the file provider, not inline Docker labels. The `ratelimit` middleware can be re-added later if needed by defining it in `config.yml`.

#### D. Add ServersTransport Configuration for Backend TLS

**Location:** In the `meshcentral` service, after line 137 (after `passHostHeader=true`)

**Add this line:**
```yaml
      - "traefik.http.services.meshcentral.loadbalancer.serversTransport=insecure@file"
```

**Full context (lines 134-138 should look like):**
```yaml
      - "traefik.http.services.meshcentral.loadbalancer.server.port=443"
      - "traefik.http.services.meshcentral.loadbalancer.server.scheme=https"
      - "traefik.http.routers.meshcentral.service=meshcentral"
      - "traefik.http.services.meshcentral.loadbalancer.passHostHeader=true"
      - "traefik.http.services.meshcentral.loadbalancer.serversTransport=insecure@file"
```

**Reason:** MeshCentral's internal TLS certificate is only valid for 127.0.0.1, not the Docker container IP (172.27.0.x). Without this, Traefik returns 500 errors due to certificate validation failures.

---

## Why These Changes Are Critical

### Problem 1: Missing File Provider Configuration
**Symptom:** Error in Traefik logs: `middleware "security-headers@file" does not exist`  
**Impact:** Traefik routes fail to load, resulting in 404 errors for all requests  
**Solution:** Add file provider configuration and mount the config file

### Problem 2: TLS Certificate Verification Failure
**Symptom:** Error in Traefik logs: `tls: failed to verify certificate: x509: certificate is valid for 127.0.0.1, not 172.27.0.4`  
**Impact:** Traefik returns 500 Internal Server Error when proxying to MeshCentral  
**Solution:** Configure `insecureSkipVerify` for internal backend connections

### Problem 3: Invalid CORS Middleware (Previously Fixed)
**Symptom:** Error: `field not found, node: cors` or `field not found, node: corsPolicy`  
**Impact:** Traefik file provider fails to load configuration  
**Solution:** CORS middleware was removed from config.yml (not valid in Traefik v3)

---

## Directory Structure Required

```
deploy/
├── docker-compose.yml          (MODIFIED - see changes above)
├── traefik/
│   └── config.yml             (NEW - create this file)
├── config.json.template        (existing)
├── init-mongo.js.template      (existing)
├── env.example                 (existing)
└── ... (other existing files)
```

---

## Deployment Instructions

When deploying or updating the application:

1. **Ensure the traefik directory exists:**
   ```bash
   mkdir -p /opt/ecortex/deploy/traefik
   ```

2. **Copy the config.yml file to the server:**
   ```bash
   scp deploy/traefik/config.yml user@server:/opt/ecortex/deploy/traefik/
   ```

3. **Apply the docker-compose.yml changes** (if updating from Git)

4. **Restart the Traefik container to pick up changes:**
   ```bash
   cd /opt/ecortex/deploy
   docker compose restart traefik
   ```

5. **If updating MeshCentral service labels, recreate the container:**
   ```bash
   docker compose up -d meshcentral
   ```

---

## Verification

After applying these changes, verify the deployment is working:

```bash
# Check Traefik logs for errors
docker logs meshcentral-traefik --tail 50 | grep -i error

# Test the website
curl -I https://ecortex.cortalis.com

# Expected result: HTTP/2 200 OK with security headers
```

---

## For Cursor AI Assistant

When updating this deployment in the future:

1. **Always preserve these files:**
   - `deploy/traefik/config.yml` (CRITICAL)

2. **Always preserve these docker-compose.yml sections:**
   - Traefik command: `--providers.file.*` lines
   - Traefik volumes: `./traefik/config.yml` mount
   - MeshCentral labels: `security-headers@file` middleware reference
   - MeshCentral labels: `serversTransport=insecure@file` configuration

3. **DO NOT:**
   - Remove or modify the `insecureSkipVerify` configuration (required for internal TLS)
   - Change `security-headers@file` back to inline labels
   - Add CORS middleware to config.yml (not compatible with Traefik v3)

4. **If adding new middleware:**
   - Add it to `deploy/traefik/config.yml` under `http.middlewares`
   - Reference it with `@file` suffix in Docker labels (e.g., `middleware-name@file`)

---

## Version Information

- **Traefik Version:** v3.0.4
- **MeshCentral Image:** ghcr.io/ylianst/meshcentral:latest
- **MongoDB Version:** 7
- **Last Updated:** 2026-01-19
- **Production Server:** ecortex.cortalis.com (3.212.168.83)
