# Git Repository Sync Summary

**Date:** 2026-01-19  
**Purpose:** Sync local Git repository with production deployment fixes

## Changes Made

This sync brings the Git repository in line with the working production deployment at `ecortex.cortalis.com`.

### New Files Added

1. **`deploy/traefik/config.yml`**
   - Traefik file provider configuration
   - Defines security-headers, ratelimit, and compression middlewares
   - Contains `insecure` serversTransport with `insecureSkipVerify: true`
   - **Critical:** Required for Traefik to route traffic correctly

2. **`deploy/DEPLOYMENT-CHANGES.md`**
   - Complete documentation of production changes
   - Explains why each change is necessary
   - Includes deployment and verification instructions
   - **For Cursor AI:** Reference guide for future updates

3. **`deploy/.cursor-rules`**
   - Quick reference card for Cursor AI
   - Lists critical configurations that must be preserved
   - Explains error symptoms and solutions

### Modified Files

1. **`deploy/docker-compose.yml`**
   
   **A. Traefik service - Added file provider:**
   ```yaml
   - "--providers.file.directory=/etc/traefik/config"
   - "--providers.file.watch=true"
   ```
   
   **B. Traefik service - Mounted config file:**
   ```yaml
   - ./traefik/config.yml:/etc/traefik/config/config.yml:ro
   ```
   
   **C. MeshCentral service - Updated middleware reference:**
   ```yaml
   # Changed from: security-headers,ratelimit
   # Changed to:   security-headers@file
   - "traefik.http.routers.meshcentral.middlewares=security-headers@file"
   ```
   
   **D. MeshCentral service - Added TLS skip verification:**
   ```yaml
   - "traefik.http.services.meshcentral.loadbalancer.serversTransport=insecure@file"
   ```

## What Problems These Changes Fix

### 1. 404 Errors (FIXED)
- **Problem:** Traefik couldn't find `security-headers@file` middleware
- **Cause:** File provider not configured, config.yml missing
- **Solution:** Added file provider + config.yml with middleware definitions

### 2. 500 Internal Server Errors (FIXED)
- **Problem:** TLS certificate verification failed when proxying to backend
- **Error:** `x509: certificate is valid for 127.0.0.1, not 172.27.0.x`
- **Cause:** MeshCentral's self-signed cert doesn't match container IP
- **Solution:** Added `insecureSkipVerify: true` for internal backend connections

### 3. Config Parse Errors (FIXED)
- **Problem:** Invalid CORS middleware syntax
- **Error:** `field not found, node: cors` or `corsPolicy`
- **Cause:** Traefik v3 doesn't support standalone CORS middleware
- **Solution:** Removed CORS config (not needed for this deployment)

## Verification

After applying these changes to production (already done), the website is working:

```bash
$ curl -I https://ecortex.cortalis.com
HTTP/2 200 
permissions-policy: geolocation=(), microphone=(), camera=()
referrer-policy: strict-origin-when-cross-origin
strict-transport-security: max-age=31536000; includeSubDomains
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
```

✅ HTTP 200 OK  
✅ Security headers applied  
✅ HTTPS with Let's Encrypt certificate  
✅ Accessible externally

## Next Steps

To commit these changes to Git:

```bash
cd /Users/clint/Documents/GitHub/eMeshCentral

# Add all new files
git add deploy/traefik/
git add deploy/DEPLOYMENT-CHANGES.md
git add deploy/.cursor-rules
git add deploy/GIT-SYNC-SUMMARY.md

# Add modified file
git add deploy/docker-compose.yml

# Commit with descriptive message
git commit -m "Fix Traefik v3 compatibility and backend TLS issues

- Add Traefik file provider configuration (traefik/config.yml)
- Configure file provider in docker-compose.yml
- Update middleware reference to use @file provider
- Add insecure serversTransport for backend TLS skip
- Document all changes for Cursor AI and future updates

Fixes:
- 404 errors (missing middleware)
- 500 errors (TLS cert verification)
- Config parse errors (invalid CORS syntax)

Production deployment verified working at ecortex.cortalis.com"

# Push to origin
git push origin master
```

## Important Notes for Cursor AI

When this repository is updated in the future:

1. **NEVER remove** `deploy/traefik/config.yml` - it's production-critical
2. **ALWAYS preserve** the file provider configuration in docker-compose.yml
3. **DO NOT change** `insecureSkipVerify: true` (required for backend TLS)
4. **Reference** `deploy/DEPLOYMENT-CHANGES.md` before making deployment changes
5. **Check** `deploy/.cursor-rules` for quick reference on critical configs

## Files to Review

- `deploy/DEPLOYMENT-CHANGES.md` - Full technical documentation
- `deploy/.cursor-rules` - Quick reference for Cursor AI
- `deploy/traefik/config.yml` - Traefik middleware and transport config
- `deploy/docker-compose.yml` - Updated with file provider support
