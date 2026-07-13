# LinView — Deployment Guide

## Architecture

```
Internet
    │
    ▼
Reverse Proxy (nginx / Caddy)   ← TLS termination
    │
    ▼
Gunicorn (4 workers)
    │
    ├── Flask app (app.py)
    │
    └── SQLite (data.db)
```

LinView is designed to run on a separate server from the audit targets. All target machines submit scans to this central dashboard via REST API.

## Production Setup

### 1. Dependencies

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2. Environment Configuration

```bash
export PORT=5000
export SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
export LINVIEW_DB=/var/lib/linview/data.db
export LINVIEW_DEBUG=false
export LINVIEW_RATE_LIMIT=60
```

### 3. Run with Gunicorn

Use the bundled `start.sh` or run directly:

```bash
gunicorn -w 4 -b 127.0.0.1:5000 app:app
```

The `start.sh` script auto-generates a `SECRET_KEY` and binds to `0.0.0.0:$PORT`.

### 4. Reverse Proxy (TLS Termination)

**Never** expose the dashboard to the internet without TLS. Use a reverse proxy.

#### nginx

```nginx
server {
    listen 443 ssl;
    server_name dash.example.com;

    ssl_certificate     /etc/letsencrypt/live/dash.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dash.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Caddy

```
dash.example.com {
    reverse_proxy 127.0.0.1:5000
}
```

### 5. Run as a System Service

Create `/etc/systemd/system/linview.service`:

```ini
[Unit]
Description=LinView
After=network.target

[Service]
Type=simple
User=linview
WorkingDirectory=/opt/linview
Environment=PORT=5000
Environment=LINVIEW_DB=/var/lib/linview/data.db
ExecStart=/opt/linview/venv/bin/gunicorn -w 4 -b 127.0.0.1:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now linview
```

## Security Hardening

### Database Location

Store the SQLite database outside the web root:

```bash
export LINVIEW_DB=/var/lib/linview/data.db
```

Ensure the directory has restricted permissions:

```bash
sudo mkdir -p /var/lib/linview
sudo chown linview:linview /var/lib/linview
sudo chmod 750 /var/lib/linview
```

### API Key Rotation

Keys are stored in the `api_keys` table. Rotate periodically:

```bash
sqlite3 /var/lib/linview/data.db \
  "DELETE FROM api_keys; INSERT INTO api_keys (key, label) VALUES ('linview-<new-token>', 'rotated');"
```

### Rate Limiting

Adjust the per-IP rate limit based on your environment size:

```bash
export LINVIEW_RATE_LIMIT=120
```

### Firewall

Restrict access to the API port to known scanner IPs if possible:

```bash
iptables -A INPUT -p tcp --dport 5000 -s <scanner-subnet> -j ACCEPT
iptables -A INPUT -p tcp --dport 5000 -j DROP
```

## Backup

The SQLite database is the only persistent state. Back it up regularly:

```bash
sqlite3 /var/lib/linview/data.db ".backup /backup/linview-$(date +%F).db"
```

## Monitoring

LinView exposes no health endpoint by default. Use a simple TCP check:

```bash
curl -f http://127.0.0.1:5000/ || alert
```
