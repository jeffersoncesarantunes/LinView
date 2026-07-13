# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| latest  | ✅       |

## Reporting a Vulnerability

This is a web dashboard for Linux kernel hardening scan reports. If you discover a security vulnerability, please do NOT open a public issue.

Contact the maintainer directly at jefferson.antunes@gmail.com with details about the issue.

We commit to acknowledging receipt within 48 hours and providing a fix timeline within 7 days.

## Known Limitations

- **Rate limiting is in-memory per-worker**: The `LINVIEW_RATE_LIMIT` is enforced per gunicorn worker process, not shared across workers. Under high load, a client could send up to `workers × RATE_LIMIT` requests per minute.
- **CORS is globally enabled**: `flask-cors` is configured without origin restrictions. In restricted networks, add explicit `origins=` to the `CORS()` call in `app.py`.
- **No authentication on read endpoints**: The dashboard homepage and scan detail pages are publicly accessible. Put them behind a VPN or a proxy-level auth (basic auth, OAuth) if needed.
- **API key in transit**: API keys are transmitted in the `X-API-Key` header. Always use TLS in production to prevent credential leakage.
- **SQLite concurrency**: SQLite with WAL mode handles concurrent reads well, but concurrent writes may cause `database is locked` errors under heavy parallel submission. The project uses SQLite exclusively; for high-throughput deployments, consider migrating to a client-server database.
- **History rewriting**: This repository's git history was cleaned with git-filter-repo. If you have an older clone, please re-clone from the latest remote.
