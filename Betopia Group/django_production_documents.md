# Betopia VPS — Django Servers

## Projects

| App | Folder | Virtual Env | Socket | Domain |
|---|---|---|---|---|
| Django Backend (limited) | `~/betopia-limited-server` | `~/betopia-limited-server/env` | `/run/limited.sock` | dashboard.betopialimited.com |
| Django Backend (daily) | `~/betopia-daily-server` | `~/betopia-daily-server/env` | `/run/daily.sock` | server.betopiadaily.shop |

---

## Key Files

| Type | Limited | Daily |
|---|---|---|
| Django Settings | `~/betopia-limited-server/config/settings.py` | `~/betopia-daily-server/config/settings.py` |
| Gunicorn Socket | `/etc/systemd/system/limited.socket` | `/etc/systemd/system/daily.socket` |
| Gunicorn Service | `/etc/systemd/system/limited.service` | `/etc/systemd/system/daily.service` |
| Nginx Config | `/etc/nginx/sites-available/limited` | `/etc/nginx/sites-available/daily` |

```bash
# ── Limited ────────────────────────────────────────────────
nano ~/betopia-limited-server/config/settings.py
sudo nano /etc/systemd/system/limited.socket
sudo nano /etc/systemd/system/limited.service
sudo nano /etc/nginx/sites-available/limited

# ── Daily ──────────────────────────────────────────────────
nano ~/betopia-daily-server/config/settings.py
sudo nano /etc/systemd/system/daily.socket
sudo nano /etc/systemd/system/daily.service
sudo nano /etc/nginx/sites-available/daily
```

---

## Database

| App | Engine | Database | User |
|---|---|---|---|
| Limited | SQLite | `db.sqlite3` | — |
| Daily | PostgreSQL | `betopia_daily` | `betopia_daily_user` |

```bash
# Access PostgreSQL (daily)
sudo -u postgres psql
\c betopia_daily
\dt
\q
```

---

## SSL Certificates

| Domain | Certificate Path |
|---|---|
| dashboard.betopialimited.com | `/etc/letsencrypt/live/dashboard.betopialimited.com/` |
| server.betopiadaily.shop | `/etc/letsencrypt/live/server.betopiadaily.shop/` |

```bash
# List all certificates
sudo certbot certificates

# Renew
sudo certbot renew --dry-run
```

---

## Gunicorn (Systemd)

```bash
# ── Status ─────────────────────────────────────────────────
sudo systemctl status limited.socket
sudo systemctl status limited.service

sudo systemctl status daily.socket
sudo systemctl status daily.service

# ── Restart (after code changes) ───────────────────────────
sudo systemctl restart limited.socket
sudo systemctl restart limited.service

sudo systemctl restart daily.socket
sudo systemctl restart daily.service

# ── Reload systemd (after editing socket/service files) ────
sudo systemctl daemon-reload
```

---

## Nginx

```bash
# Test config
sudo nginx -t

# Reload (preferred)
sudo systemctl reload nginx

# Full restart
sudo systemctl restart nginx

# List sites
ls /etc/nginx/sites-enabled/
ls /etc/nginx/sites-available/
```

---

## Static Files

```bash
# ── Limited ────────────────────────────────────────────────
source ~/betopia-limited-server/env/bin/activate
cd ~/betopia-limited-server
python manage.py collectstatic
sudo chmod -R 755 /home/ubuntu/betopia-limited-server/staticfiles
sudo chmod o+rx /home/ubuntu /home/ubuntu/betopia-limited-server

# ── Daily ──────────────────────────────────────────────────
source ~/betopia-daily-server/env/bin/activate
cd ~/betopia-daily-server
python manage.py collectstatic
sudo chmod -R 755 /home/ubuntu/betopia-daily-server/staticfiles
sudo chmod o+rx /home/ubuntu /home/ubuntu/betopia-daily-server
```

---

## After VPS Reboot

Both services are enabled on startup. If they don't come up:

```bash
sudo systemctl start limited.socket
sudo systemctl start limited.service

sudo systemctl start daily.socket
sudo systemctl start daily.service

sudo systemctl restart nginx
```

Verify:

```bash
sudo systemctl status limited.service
sudo systemctl status daily.service
sudo systemctl status nginx
```

---

## Debug

```bash
# ── Gunicorn logs ──────────────────────────────────────────
sudo journalctl -u limited.service -n 50
sudo journalctl -u daily.service -n 50

# ── Nginx error log ────────────────────────────────────────
sudo tail -f /var/log/nginx/error.log

# ── Check socket files exist ───────────────────────────────
file /run/limited.sock
file /run/daily.sock
```

---

## Test URLs

```
# Limited
https://dashboard.betopialimited.com/admin/
https://dashboard.betopialimited.com/static/admin/css/base.css

# Daily
https://server.betopiadaily.shop/admin/
https://server.betopiadaily.shop/static/admin/css/base.css
```

---

## Full Request Flow

```
Browser
   ↓
Nginx (HTTPS)
   ↓
/static → staticfiles/
/media  → media/
/       → .sock
   ↓
Gunicorn
   ↓
Django (config.wsgi)
   ↓
Database (SQLite / PostgreSQL)
```

---

## Notes

- Each Django project needs its own socket, service, and nginx config
- `.socket` = systemd configuration file
- `.sock` = runtime Unix socket file created at startup
- Nginx serves static/media directly — never hits Gunicorn
- PostgreSQL 15+ requires `GRANT ALL ON SCHEMA public` for new users