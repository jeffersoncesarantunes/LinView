# LinView — Architecture

## Overview

LinView is a lightweight Flask web application that receives kernel hardening scan reports from [LinSpec](https://github.com/jeffersoncesarantunes/LinSpec) via REST API and displays them in a web interface.

## Role in the SYNTROPY Ecosystem

```
┌─────────────────────────────────────────────────────┐
│                    SYNTROPY                         │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌────────────┐       │
│  │ LinSpec  │──>│  SIREN   │──>│ K-Scanner  │       │
│  │ (audit)  │   │(acquire) │   │ (analyze)  │       │
│  └────┬─────┘   └──────────┘   └────────────┘       │
│       │                                             │
│       └────────────────────────────┐                │
│                                    ▼                │
│  ┌──────────────────────────────┐                   │
 │  │    LinView                    │                   │
 │  │  (visualization & history)   │                   │
│  └──────────────────────────────┘                   │
└─────────────────────────────────────────────────────┘
```

LinSpec runs on the target machine and outputs a `report.json`. LinView receives this report, stores it in SQLite, and provides:

- **Aggregate view**: average score, total PASS/WARN/VULN across all scans
- **Per-scan detail**: individual check results per host
- **Historical tracking**: all scans ordered by date

## Data Flow

```
LinSpec (target machine)
    │
    │  POST /api/scan (JSON + X-API-Key)
    ▼
Flask app (dashboard server)
    │
    ├── SQLite DB (scans, checks, api_keys)
    │
    └── Jinja2 templates → HTML dashboard
```

## API

| Method | Route | Auth | Description |
|---|---|---|---|
| GET | `/` | No | Dashboard homepage |
| GET | `/scan/<id>` | No | Per-scan detail page |
| GET | `/api/scan/<id>/raw` | No | Raw JSON of a scan |
| POST | `/api/scan` | X-API-Key | Submit a scan report |
| POST | `/api/subscribe` | No | Newsletter subscription |
| GET | `/admin/setup` | No | Generate first API key |

## Security

- API key authentication on `/api/scan` (subscribe is rate-limited only)
- Rate limiting (60 req/min per IP, configurable via `LINVIEW_RATE_LIMIT`)
- SQL injection prevented via parameterized queries
- Debug mode disabled by default (`LINVIEW_DEBUG=false`)
- DB path configurable via `LINVIEW_DB` environment variable

## Deployment

The dashboard is designed to run on a separate machine from the audit targets. See [README](../README.md) for production setup with gunicorn.
