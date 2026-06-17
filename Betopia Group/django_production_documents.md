# Betopia VPS — Django Daily Server

## Project

| App | Folder | Virtual Env | Socket | Domain |
|---|---|---|---|---|
| Django Backend (daily) | `~/betopia-daily-server` | `~/betopia-daily-server/env` | `/run/daily.sock` | server.betopiadaily.shop |

---

## Project Structure

```
betopia-daily-server/
├── apps/
├── config/            # settings.py, wsgi.py
├── staticfiles/       # collectstatic output
├── media/             # uploaded files
├── env/               # virtual environment
├── manage.py
└── db.sqlite3         # replaced by PostgreSQL
```

---

## Key Files

| Type | File |
|---|---|
| Django Settings | `~/betopia-daily-server/config/settings.py` |
| Gunicorn Socket | `/etc/systemd/system/daily.socket` |
| Gunicorn Service | `/etc/systemd/system/daily.service` |
| Nginx Config | `/etc/nginx/sites-available/daily` |

```bash
# Open Django settings
nano ~/betopia-daily-server/config/settings.py

# Open Gunicorn socket
sudo nano /etc/systemd/system/daily.socket

# Open Gunicorn service
sudo nano /etc/systemd/system/daily.service

# Open Nginx config
sudo nano /etc/nginx/sites-available/daily
```

---

## Database (PostgreSQL)

| Key | Value |
|---|---|
| Database | `betopia_daily` |
| User | `betopia_daily_user` |
| Host | `localhost` |
| Port | `5432` |

```bash
# Access PostgreSQL
sudo -u postgres psql

# Connect to the database
\c betopia_daily

# List tables
\dt

# Exit
\q
```

---

## SSL Certificate

| Domain | Certificate Path |
|---|---|
| server.betopiadaily.shop | `/etc/letsencrypt/live/server.betopiadaily.shop/` |

```bash
# List all certificates
sudo certbot certificates

# Renew certificates
sudo certbot renew --dry-run
```

---

## Gunicorn (Systemd)

```bash
# Status
sudo systemctl status daily.socket
sudo systemctl status daily.service

# Restart (after code changes)
sudo systemctl restart daily.socket
sudo systemctl restart daily.service

# Reload systemd (after editing socket/service files)
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

# List enabled sites
ls /etc/nginx/sites-enabled/

# List available sites
ls /etc/nginx/sites-available/
```

---

## Static Files

```bash
# Activate venv
source ~/betopia-daily-server/env/bin/activate

# Collect static (run after UI changes)
cd ~/betopia-daily-server
python manage.py collectstatic

# Fix permissions if 403 errors appear
sudo chmod -R 755 /home/ubuntu/betopia-daily-server/staticfiles
sudo chmod o+rx /home/ubuntu /home/ubuntu/betopia-daily-server
```

---

## After VPS Reboot

Gunicorn socket and service are enabled on startup. If they don't come up:

```bash
sudo systemctl start daily.socket
sudo systemctl start daily.service
sudo systemctl restart nginx
```

Verify everything is running:

```bash
sudo systemctl status daily.service
sudo systemctl status daily.socket
sudo systemctl status nginx
```

---

## Debug

```bash
# Gunicorn logs
sudo journalctl -u daily.service -n 50

# Nginx error log
sudo tail -f /var/log/nginx/error.log

# Check socket file exists
file /run/daily.sock
```

---

## Test URLs

```
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
/       → /run/daily.sock
   ↓
Gunicorn
   ↓
Django (config.wsgi)
   ↓
PostgreSQL (betopia_daily)
```