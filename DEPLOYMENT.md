# Phase 2: Traefik Integration & Deployment Guide

## Overview

This guide walks through deploying the multi-tenant authentication system with Traefik as the reverse proxy and load balancer.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          TRAEFIK (Reverse Proxy)                │
│                         Port 80 → 443 (TLS)                      │
├─────────────────────────────────────────────────────────────────┤
│                         traefik-public network                   │
├──────────────────┬──────────────────────┬───────────────────────┤
│                  │                      │                       │
│   Auth Manager   │    Dashboard (User1) │   Dashboard (User2)  │
│  :8080/auth      │   :port-1/           │   :port-2/           │
│  :8080/login     │                      │                      │
│                  │    API (User1)       │   API (User2)        │
│                  │   :port-1/api        │   :port-2/api        │
│                  │                      │                      │
└──────────────────┴──────────────────────┴───────────────────────┘
         ↓ (internal)       ↓ (internal)        ↓ (internal)
   auth-internal    user1-tenant1-internal  user2-tenant1-internal
        │                  │                     │
    Skytable          PostgreSQL              PostgreSQL
    :2003            (user1 data)            (user2 data)
     Redis
```

## Core Services (Shared)

These services run once and are shared across all users:

### 1. Skytable (Database for Auth)
- **Role**: Session storage, login token verification, rate limiting
- **Port**: 2003 (internal to auth-internal network)
- **Persistence**: AOF + snapshots to `./skydb/`
- **Credentials**: Set in `docker-compose.yml`

### 2. Auth Manager (Rust HTTP Server)
- **Role**: Handles `/api/auth/login` and `/auth` (ForwardAuth) endpoints
- **Port**: 8080 (internal to traefik-public)
- **Routes**:
  - `POST /api/auth/login` - Create session from token
  - `GET /auth` - ForwardAuth validation (Traefik calls this)
  - `GET /health` - Health check
- **Middleware**: Protected by Traefik's auth-forward
- **Build**: Multi-stage Dockerfile (Rust → debian:bookworm-slim)

### 3. Traefik (Reverse Proxy)
- **Role**: HTTP routing, TLS termination, ForwardAuth enforcement
- **Port**: 80 (HTTP), 443 (HTTPS/TLS)
- **Config**:
  - Static: `traefik/traefik.yml` (entry points, providers, SSL)
  - Dynamic: `traefik/dynamic-conf.yml` (middlewares, routers, services)
- **Features**:
  - Docker provider (auto-discover containers)
  - Cloudflare DNS-01 challenge for wildcard SSL
  - Zero-expose by default (containers must opt-in with `traefik.enable=true`)
  - Custom middleware: `auth-forward` (calls Auth Manager's `/auth` endpoint)

## Per-User Stacks

Each user gets their own isolated stack:

```yaml
stack: user1-tenant1
├── dashboard        (SPA frontend, protected by auth-forward)
├── public-api      (REST API, protected by auth-forward)
├── database        (PostgreSQL, isolated network)
└── redis           (session cache, isolated network)
```

### Routes Created
- **Dashboard**: `user1-tenant1.yourdomain.com` → dashboard:80
- **API**: `api-user1-tenant1.yourdomain.com` → public-api:3000

Both routes protected by `auth-forward` middleware.

---

## Deployment Steps

### Step 1: Start Core Services

```bash
cd /home/pilser/TCPMemoryServer/UsersManagers
docker compose -f docker/docker-compose.yml up -d
```

**Verify:**
```bash
docker ps
# Should show: skytable, auth-manager, traefik

docker compose -f docker/docker-compose.yml logs -f auth-manager
# Should see "listening on 0.0.0.0:8080"
```

### Step 2: Test Auth Flow

```bash
./scripts/test-integration.sh
```

**Expected output:**
```
✅ Skytable is running
✅ Auth Manager is running
✅ Traefik is running
✅ Health endpoint responding
✅ Login flow working (session created)
✅ ForwardAuth validation working (user headers returned)
```

### Step 3: Deploy User Stack (Example)

```bash
./scripts/deploy-user-stack.sh user1 tenant1 yourdomain.com
```

**What happens:**
1. Creates `docker/user-stacks/user1-tenant1-compose.yml` from template
2. Generates `docker/user-stacks/user1-tenant1.env` with random DB password + JWT secret
3. Creates isolated networks: `user1-tenant1-internal`
4. Spawns 4 services: dashboard, API, PostgreSQL, Redis
5. Labels services for Traefik routing

**Verify:**
```bash
docker ps | grep user1
# Should show: user1-tenant1-dashboard, user1-tenant1-public-api, 
#              user1-tenant1-database, user1-tenant1-redis

docker compose -f docker/user-stacks/user1-tenant1-compose.yml logs -f
```

### Step 4: Configure Domain/DNS

**Option A: Local Testing (with /etc/hosts)**
```bash
sudo tee -a /etc/hosts << EOF
127.0.0.1 yourdomain.com
127.0.0.1 user1-tenant1.yourdomain.com
127.0.0.1 api-user1-tenant1.yourdomain.com
127.0.0.1 auth.yourdomain.com
EOF
```

**Option B: Production (with Cloudflare)**
1. Add DNS records to Cloudflare:
   - `*.yourdomain.com` → Your server IP
   - `yourdomain.com` → Your server IP
2. Cloudflare will auto-resolve wildcard

3. In `traefik/traefik.yml`, ensure certificate resolver is configured:
   ```yaml
   certificatesResolvers:
     cloudflare:
       acme:
         email: your-email@example.com
         storage: /certs/acme.json
         dnsChallenge:
           provider: cloudflare
   ```

### Step 5: Test User Access

```bash
# Through Traefik (via hostname routing)
curl -i https://user1-tenant1.yourdomain.com/
```

**Expected flow:**
1. Traefik receives request to `user1-tenant1.yourdomain.com`
2. Traefik calls `auth-forward` middleware
3. Middleware calls Auth Manager's `/auth` endpoint with session cookie
4. Auth Manager validates session against Skytable
5. If valid, returns `X-Auth-User: user1`, `X-Auth-Tenant: tenant1` headers
6. Traefik forwards request to dashboard service with headers
7. Dashboard receives user info via headers

---

## Configuration Reference

### Environment Variables (Core Services)

**Skytable**
```env
SKYTABLE_HOST=skytable
SKYTABLE_PORT=2003
SKYTABLE_USERNAME=root
SKYTABLE_PASSWORD=root
```

**Auth Manager**
```env
SERVER_PORT=8080
SKYTABLE_HOST=skytable
SKYTABLE_PORT=2003
SKYTABLE_USERNAME=root
SKYTABLE_PASSWORD=root
SESSION_TIMEOUT_MINUTES=60
TOKEN_EXPIRY_DAYS=365
```

### Per-User Stack Variables

Generated by `deploy-user-stack.sh`:
```env
USER_ID=user1
TENANT_ID=tenant1
STACK_NAME=user1-tenant1
DOMAIN_NAME=yourdomain.com
DB_USER=app_user1
DB_PASSWORD=<random>
DB_NAME=user_user1
JWT_SECRET=<random>
```

---

## Monitoring & Troubleshooting

### View Logs

```bash
# Core services
docker compose -f docker/docker-compose.yml logs -f

# Specific service
docker compose -f docker/docker-compose.yml logs -f traefik
docker compose -f docker/docker-compose.yml logs -f auth-manager

# User stack
docker compose -f docker/user-stacks/user1-tenant1-compose.yml logs -f
```

### Check Traefik Dashboard

```
http://localhost:8081/dashboard/
```

Shows:
- HTTP routers and their status
- Services and endpoints
- Middleware definitions
- TLS certificates

### Common Issues

**Issue: 502 Bad Gateway from Traefik**
- Check Auth Manager is running: `curl http://localhost:8080/health`
- Check Traefik can reach Auth Manager: `docker logs docker-traefik-1`
- Verify service is on `traefik-public` network

**Issue: Auth failing (401 Unauthorized)**
- Check session is valid: `docker logs docker-auth-manager-1`
- Verify Skytable has session: `docker exec docker-skytable-1 skysh`
- Run test script: `./scripts/test-integration.sh`

**Issue: TLS certificate not generated**
- Check Traefik logs: `docker logs docker-traefik-1`
- Ensure DNS is resolving
- If using self-signed for testing, disable `tls.insecure=true` in traefik.yml

**Issue: User stack services can't reach Skytable**
- Verify `traefik-public` network is connected
- Auth Manager must be reachable at `http://auth-manager:8080` from within stack

---

## Scaling Guide

### Add New User

```bash
./scripts/deploy-user-stack.sh user2 tenant1 yourdomain.com
./scripts/deploy-user-stack.sh user3 tenant2 yourdomain.com
```

Each gets:
- Isolated database (PostgreSQL)
- Isolated cache (Redis)
- Isolated network
- Routes through Traefik with auth protection

### Load Balancing

Traefik automatically distributes traffic:
```yaml
traefik.http.services.user1-api.loadbalancer.server.port=3000
```

To scale API replicas:
```bash
docker compose -f docker/user-stacks/user1-tenant1-compose.yml up -d --scale public-api=3
```

Traefik discovers all 3 instances and load-balances.

---

## Security Checklist

- [ ] Generate strong DB passwords (not defaults)
- [ ] Generate strong JWT secrets
- [ ] Enable TLS (Cloudflare or Let's Encrypt)
- [ ] Set `Cookie: Secure, HttpOnly, SameSite=Strict`
- [ ] Implement rate limiting on Auth Manager
- [ ] Use network policies to isolate services
- [ ] Enable Traefik API authentication if exposing dashboard
- [ ] Rotate session secrets regularly
- [ ] Monitor failed login attempts

---

## Next: Phase 3 - Multi-Tenant Isolation

Once Traefik routing is verified, Phase 3 will:
1. Enforce tenant_id isolation in database queries
2. Verify user only accesses their tenant's data
3. Implement RBAC (role-based access control)
4. Add audit logging for tenant operations

See `Basic plan for multi users.md` for full roadmap.
