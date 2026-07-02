# LinSpec Dashboard

Web dashboard to receive and display LinSpec kernel hardening scan reports.

[![Platform-Linux](https://img.shields.io/badge/Platform-Linux-1793D1?style=flat-square&logo=linux&logoColor=white)](https://kernel.org)
[![Language-Python](https://img.shields.io/badge/Language-Python-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Framework-Flask](https://img.shields.io/badge/Framework-Flask-000000?style=flat-square&logo=flask&logoColor=white)](https://flask.palletsprojects.com)
[![License-MIT](https://img.shields.io/badge/License-MIT-EE0000?style=flat-square&logo=license&logoColor=white)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Active-00A86B?style=flat-square)](#-roadmap)
[![CI](https://img.shields.io/github/actions/workflow/status/jeffersoncesarantunes/LinDash/ci.yml?style=flat-square&logo=github&label=CI)](https://github.com/jeffersoncesarantunes/LinDash/actions/workflows/ci.yml)
[![Tested-on](https://img.shields.io/badge/Tested%20on-Arch%20Linux-1793D1?style=flat-square&logo=arch-linux)](https://security.archlinux.org/)
[![Domain](https://img.shields.io/badge/Domain-Security%20Dashboard-8A2BE2?style=flat-square)](docs/ARCHITECTURE.md)

## Overview

Collects scan reports via REST API, stores them in SQLite, and displays aggregate statistics and per-scan details. Natively compatible with LinSpec JSON output via `--webhook`.

## Quick Start

```bash
pip install -r requirements.txt
python app.py
```

Open http://localhost:5000

### Generate an API key

Visit `/admin/setup` to create the first API key for submitting scans.

## API

### Submit a scan report

Accepts both **LinSpec-native** format (lowercase `result`) and **legacy** format (uppercase `status`). Examples:

```bash
# LinSpec-native format (from --webhook)
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

# Legacy format
curl -X POST http://localhost:5000/api/scan \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-key>" \
  -d '{
    "hostname": "server01",
    "checks": [
      {"check": "aslr", "category": "memory", "status": "PASS", "message": ""},
      {"check": "kptr_restrict", "category": "kernel", "status": "VULN", "message": "bad"}
    ]
  }'
```

### View a raw scan

```
GET /api/scan/<id>/raw
```

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `5000` | HTTP port |
| `SECRET_KEY` | auto | Flask session secret |
| `LINSPEC_DB` | `data.db` | SQLite database path |
| `LINSPEC_DEBUG` | `false` | Enable Flask debug mode |
| `LINSPEC_RATE_LIMIT` | `60` | Max requests per minute per IP |

## Security Notes

- **API key** is read from the `X-API-Key` header only (query string `?key=` not accepted)
- **CORS** is enabled globally via `flask-cors`
- **Rate limiting** is in-memory per-worker (not shared across gunicorn workers)
- Always use a reverse proxy (nginx/caddy) for TLS termination

## Production

Use the bundled `start.sh` with gunicorn:

```bash
./start.sh
```

**Never** run with `LINSPEC_DEBUG=true` in production.

## Tests

```bash
pip install pytest
python -m pytest tests/
```

