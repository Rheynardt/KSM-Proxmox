# KSM Proxmox

Self-hosted middleware infrastructure for New Zealand business integrations. Runs on Proxmox VE with three LXC containers that bridge Power Automate workflows to government and accounting APIs.

## Why This Exists

Several NZ business APIs require capabilities that Power Automate cannot provide directly:

| API | Challenge | Solution |
|-----|-----------|----------|
| **IRD Gateway** | Requires mTLS client certificates | MyIRD container handles certificate auth |
| **Companies Office** | Uses Azure B2C OAuth with browser consent | CompaniesOffice container manages token lifecycle |
| **Xero Practice Manager** | Complex OAuth2 + needs data warehouse | Xero-Sync syncs to Supabase for reporting |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              POWER AUTOMATE                                  │
│                         (Microsoft Cloud Flows)                              │
└──────────────┬───────────────────┬───────────────────┬──────────────────────┘
               │                   │                   │
               ▼                   ▼                   ▼
        ┌──────────┐        ┌──────────┐        ┌──────────┐
        │Cloudflare│        │Cloudflare│        │  Direct  │
        │ Tunnel   │        │ Tunnel   │        │  (Cron)  │
        └────┬─────┘        └────┬─────┘        └────┬─────┘
             │                   │                   │
═════════════╪═══════════════════╪═══════════════════╪════════════════════════
             │              PROXMOX VE HOST          │
             │           192.168.61.0/24             │
             ▼                   ▼                   ▼
┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
│   LXC 102          │ │   LXC 101          │ │   LXC 100          │
│   CompaniesOffice  │ │   MyIRD            │ │   Xero-Sync        │
│   .225             │ │   .226             │ │   .228             │
│                    │ │                    │ │                    │
│ ┌────────────────┐ │ │ ┌────────────────┐ │ │ ┌────────────────┐ │
│ │ FastAPI        │ │ │ │ Docker Stack   │ │ │ │ Python Script  │ │
│ │ + OAuth B2C    │ │ │ │ + mTLS Certs   │ │ │ │ + OAuth2       │ │
│ └────────────────┘ │ │ └────────────────┘ │ │ └────────────────┘ │
└─────────┬──────────┘ └─────────┬──────────┘ └─────────┬──────────┘
          │                      │                      │
          ▼                      ▼                      ▼
┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
│  Companies Office  │ │   IRD Gateway      │ │ Xero Practice Mgr  │
│  API (MBIE)        │ │   Services         │ │ + Supabase         │
└────────────────────┘ └────────────────────┘ └────────────────────┘
```

## Containers

### Container 100: Xero-Sync
**Purpose**: Synchronizes Xero Practice Manager data to a Supabase PostgreSQL database for reporting and analytics.

| Property | Value |
|----------|-------|
| IP Address | 192.168.61.228 |
| Memory | 512 MB |
| Storage | 8 GB |
| Schedule | Daily at 02:00 UTC |

**What it syncs**:
- Staff, Clients, Jobs, Tasks
- Invoices and Time Entries (last 180 days)

**Data flow**: Xero XPM API → Python sync script → Supabase

---

### Container 101: MyIRD
**Purpose**: Proxy for IRD Gateway Services that handles mTLS certificate authentication. Power Automate cannot use client certificates directly.

| Property | Value |
|----------|-------|
| IP Address | 192.168.61.226 |
| Memory | 1 GB |
| Storage | 8 GB |
| Runtime | Docker Compose |

**Stack**:
- FastAPI app with mTLS client certificates
- Caddy reverse proxy
- Cloudflare tunnel for external access

**Security**:
- API key required for all requests
- Only proxies paths starting with `gateway/`
- Client certificates stored in `/certs`

---

### Container 102: CompaniesOffice
**Purpose**: Middleware for the NZ Companies Office API. Handles the Azure B2C OAuth flow that requires browser-based consent.

| Property | Value |
|----------|-------|
| IP Address | 192.168.61.225 |
| Memory | 4 GB |
| Storage | 8 GB |
| Runtime | systemd service |

**Capabilities**:
- Company search and registration
- Director and shareholder management
- Annual returns submission
- Tax and GST registration

**Auth flow**:
1. Call `/oauth/authorize` to get authorization URL
2. Complete consent in browser
3. Callback stores tokens automatically
4. Tokens auto-refresh on subsequent requests

---

## Network

All containers run on the `vmbr0` bridge network:

| Container | Hostname | IP Address |
|-----------|----------|------------|
| 100 | Xero-Sync | 192.168.61.228 |
| 101 | MyIRD | 192.168.61.226 |
| 102 | CompaniesOffice | 192.168.61.225 |

Gateway: `192.168.61.254`

## External Dependencies

| Service | Purpose |
|---------|---------|
| [Supabase](https://supabase.com) | PostgreSQL database for XPM data warehouse |
| [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) | Secure ingress without port forwarding |
| [Xero Developer](https://developer.xero.com) | OAuth2 credentials for Practice Manager API |
| [IRD Gateway](https://www.ird.govt.nz/digital-service-providers/services-capability/gateway-services) | Government tax services API |
| [Companies Office API](https://api.business.govt.nz) | MBIE company registration API |

## Quick Reference

```bash
# List all containers
pct list

# Enter a container
pct enter 100

# Check container status
pct status 100

# View logs
pct exec 100 -- tail -100 /opt/xero-sync/cron.log
pct exec 101 -- docker logs myird-app-1 --tail 100
pct exec 102 -- journalctl -u companies-office-middleware -n 100
```

## License

Private infrastructure configuration. Not intended for public use.
