# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment Overview

This is a Proxmox VE 8 host running three LXC containers for NZ business integrations:

| VMID | Name | IP | Purpose |
|------|------|-----|---------|
| 100 | Xero-Sync | 192.168.61.228 | Xero Practice Manager → Supabase sync |
| 101 | MyIRD | 192.168.61.226 | IRD Gateway Services proxy (mTLS) |
| 102 | CompaniesOffice | 192.168.61.225 | Companies Office API middleware |

## Container Management

```bash
# List containers
pct list

# Enter container shell
pct enter 100

# Execute command in container
pct exec 100 -- <command>

# Start/stop/restart
pct start 100
pct stop 100
pct restart 100
```

---

## Container 100: Xero-Sync

**Location**: `/opt/xero-sync/`

Python service that syncs Xero Practice Manager data to Supabase on a daily cron schedule.

### Commands

```bash
# Run sync manually
pct exec 100 -- /opt/xero-sync/venv/bin/python /opt/xero-sync/sync_xpm.py

# View logs
pct exec 100 -- tail -100 /opt/xero-sync/cron.log

# Trigger via HTTP
pct exec 100 -- curl -X POST http://127.0.0.1:8787/run-sync
```

### Architecture

```
Xero Practice Manager API (XML/JSON) → sync_xpm.py → Supabase PostgreSQL
```

**Key Files**:
- `sync_xpm.py` - Main sync with OAuth2, API calls, data mappers
- `trigger_server.py` - HTTP trigger server (port 8787)
- `xero_token.json` - OAuth2 tokens (auto-refreshed)

**Schedule**: Daily at 02:00 UTC via cron

**Synced Tables**: Staff, Clients, Jobs (3-year chunks), Tasks, Invoices (180 days), TimeEntries (180 days)

**XPM Quirks**: No pagination support, requires yearly date chunking for Jobs, mixed XML/JSON responses.

---

## Container 101: MyIRD

**Location**: `/root/MyIRD/`

FastAPI proxy for NZ Inland Revenue Department Gateway Services. Handles mTLS certificate auth and OAuth2 tokens. Runs in Docker with Caddy reverse proxy and Cloudflare tunnel.

### Commands

```bash
# View running containers
pct exec 101 -- docker ps

# View logs
pct exec 101 -- docker logs myird-app-1 --tail 100

# Restart stack
pct exec 101 -- bash -c 'cd /root/MyIRD && docker compose restart'

# Rebuild after code changes
pct exec 101 -- bash -c 'cd /root/MyIRD && docker compose up -d --build'
```

### Architecture

```
Power Automate → Cloudflare Tunnel → Caddy (port 8080) → FastAPI (port 8000) → IRD Gateway (mTLS)
```

**Docker Stack**:
- `myird-app-1` - FastAPI app with mTLS client certs
- `myird-caddy-1` - Reverse proxy
- `myird-tunnel-1` - Cloudflare tunnel

**Key Files**:
- `app/main.py` - FastAPI app setup
- `app/ird_client.py` - mTLS client with auto token refresh
- `app/routers/ird_proxy.py` - Transparent proxy (only allows `gateway/*` paths)
- `app/config.py` - Pydantic settings from `.env`
- `certs/` - Client certificates for IRD mTLS

**Security**: Requires `X-API-Key` header. Only proxies paths starting with `gateway/`.

---

## Container 102: CompaniesOffice

**Location**: `/opt/companies-office-middleware/`

FastAPI middleware for NZ Companies Office API. Handles OAuth2 B2C flow and provides simplified endpoints for company operations.

### Commands

```bash
# View service status
pct exec 102 -- systemctl status companies-office-middleware

# View logs
pct exec 102 -- journalctl -u companies-office-middleware -n 100

# Restart service
pct exec 102 -- systemctl restart companies-office-middleware

# Check OAuth status
pct exec 102 -- curl http://localhost:8000/health
```

### Architecture

```
Power Automate → Cloudflare Tunnel → FastAPI (port 8000) → Companies Office API (B2C OAuth)
```

**Key Files**:
- `main.py` - FastAPI app with all endpoints
- `oauth.py` - TokenManager for B2C OAuth flow
- `config.py` - Settings from `.env`
- `tokens.json` - Persisted OAuth tokens

**Endpoints**:
- `/health` - Check auth status
- `/oauth/authorize` - Start OAuth flow
- `/oauth/callback` - OAuth callback
- `/companies` - Search/create companies
- `/companies/{uuid}` - Company CRUD
- `/companies/{uuid}/directors` - Director management
- `/companies/{uuid}/shareholding` - Shareholding management
- `/companies/{uuid}/annual-returns` - Annual return submission
- `/companies/{uuid}/tax-registration` - Tax registration
- `/companies/{uuid}/gst-registration` - GST registration

**Security**: Requires `X-API-Key` header matching `.env` value.

---

## Network

All containers are on `vmbr0` bridge with gateway `192.168.61.254`. Each has:
- SSH access enabled
- Postfix for mail
- Firewall enabled

## Backups

Container backups should be configured in Proxmox Backup Server or via `vzdump`.
