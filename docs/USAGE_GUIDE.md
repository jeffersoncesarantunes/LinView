# LinSpec Dashboard — Usage Guide

## Overview

LinDash is a web dashboard that receives kernel hardening scan reports from [LinSpec](https://github.com/jeffersoncesarantunes/LinSpec) and provides a centralized view of security posture across all your hosts.

## Quick Start

### 1. Start the Dashboard

```bash
pip install -r requirements.txt
python app.py
```

The dashboard is now running at `http://localhost:5000`.

### 2. Generate an API Key

Visit `http://localhost:5000/admin/setup` to create the first API key.

The key is a one-time generated token. Store it securely — it will be used to authenticate scan submissions.

### 3. Submit a Scan from LinSpec

On any target machine with LinSpec installed:

```bash
linspec --webhook http://<dashboard-ip>:5000/api/scan --api-key <your-key>
```

You can also submit manually via curl:

```bash
curl -X POST http://localhost:5000/api/scan \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-key>" \
  -d '{
    "hostname": "server01",
    "kernel": "6.8.0",
    "os": "Linux",
    "checks": [
      {"id": 1, "name": "aslr", "result": "pass", "category": "memory",
       "current": 2, "expected": 2, "message": ""},
      {"id": 2, "name": "kptr_restrict", "result": "vuln", "category": "kernel",
       "current": 0, "expected": 2, "message": "kptr_restrict=0"}
    ]
  }'
```

### 4. View Results

Open `http://localhost:5000` in your browser.

## Dashboard Walkthrough

### Home Page

The dashboard displays:

- **Stat cards** at the top: total scans, subscriber count, average score, and aggregate PASS/WARN/VULN counts
- **Scans table** listing the most recent 50 scans with hostname, kernel, check counts, and score

### Score Interpretation

| Score | Color   | Meaning                              |
|-------|---------|--------------------------------------|
| ≥80%  | Green   | Good security posture                |
| ≥50%  | Yellow  | Moderate risk, attention needed      |
| <50%  | Red     | Critical, immediate remediation advised |

### Per-Scan Detail

Click any hostname in the table to view:

- Full breakdown of every check: name, category, status, and message
- Color-coded status badges (PASS/WARN/VULN)
- Score bar with visual severity indicator
- Link to raw JSON for programmatic access

### Raw JSON Access

Each scan's raw payload is available at:

```
GET /api/scan/<id>/raw
```

## Managing API Keys

### Initial Setup

Visit `/admin/setup` on a fresh installation to auto-generate the first key.

### Manual Management

Keys are stored in the `api_keys` table of the SQLite database. To add or revoke keys manually:

```bash
sqlite3 data.db "INSERT INTO api_keys (key, label) VALUES ('linspec-mykey', 'workstation-prod');"
sqlite3 data.db "DELETE FROM api_keys WHERE key='linspec-mykey';"
```

## Supported Formats

LinDash accepts both **LinSpec-native** and **legacy** formats interchangeably:

| Field      | LinSpec-native | Legacy         |
|------------|---------------|----------------|
| Check name | `name`        | `check`        |
| Status     | `result`      | `status`       |
| Values     | `pass/warn/vuln/skip/error` | `PASS/WARN/VULN/SKIP/ERROR` |

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `5000` | HTTP port |
| `SECRET_KEY` | auto | Flask session secret |
| `LINSPEC_DB` | `data.db` | SQLite database path |
| `LINSPEC_DEBUG` | `false` | Enable Flask debug mode |
| `LINSPEC_RATE_LIMIT` | `60` | Max requests per minute per IP |
