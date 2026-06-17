# Betopia Daily — Complete Deployment & Migration Guide

## Project Overview

| App | Folder | Virtual Env | Socket | Domain |
|---|---|---|---|---|
| Django Backend (daily) | `~/betopia-daily-server` | `~/betopia-daily-server/env` | `/run/daily.sock` | server.betopiadaily.shop |

---

## Project Structure

```
betopia-daily-server/
│
├── apps/
├── config/               # Django project (settings, wsgi)
├── staticfiles/          # collectstatic output
├── media/                # uploaded files
├── env/                  # virtual environment
├── manage.py
└── db.sqlite3            # replaced by PostgreSQL
```

---

## Systemd & Nginx Files Reference

| Type    | File                                  |
|---------|---------------------------------------|
| Socket  | `/etc/systemd/system/daily.socket`    |
| Service | `/etc/systemd/system/daily.service`   |
| Nginx   | `/etc/nginx/sites-available/daily`    |

---

---

# PART 1 — Initial Deployment (Gunicorn + Nginx + Systemd)

---

## Step 1 — Create Virtual Environment & Install Dependencies

```bash
cd ~/betopia-daily-server
python3 -m venv env
source env/bin/activate
pip install -r requirements.txt
pip install gunicorn
```

---

## Step 2 — Update Django Settings

```bash
nano config/settings.py
```

Add/update:

```python
ALLOWED_HOSTS = [
    "server.betopiadaily.shop",
    "www.server.betopiadaily.shop"
]

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

DEBUG = False
```

---

## Step 3 — Collect Static Files (fixes Django admin CSS)

```bash
python manage.py collectstatic
```

---

## Step 4 — Create Gunicorn Socket

```bash
sudo nano /etc/systemd/system/daily.socket
```

```ini
[Unit]
Description=betopia daily socket

[Socket]
ListenStream=/run/daily.sock

[Install]
WantedBy=sockets.target
```

---

## Step 5 — Create Gunicorn Service

```bash
sudo nano /etc/systemd/system/daily.service
```

```ini
[Unit]
Description=betopia daily gunicorn service
Requires=daily.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/betopia-daily-server

ExecStart=/home/ubuntu/betopia-daily-server/env/bin/gunicorn \
    --workers 3 \
    --access-logfile - \
    --bind unix:/run/daily.sock \
    config.wsgi:application

[Install]
WantedBy=multi-user.target
```

---

## Step 6 — Create Nginx Config

```bash
sudo nano /etc/nginx/sites-available/daily
```

```nginx
server {
    listen 80;
    server_name server.betopiadaily.shop www.server.betopiadaily.shop;

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location /static/ {
        alias /home/ubuntu/betopia-daily-server/staticfiles/;
        autoindex off;
        expires 30d;
        add_header Cache-Control "public";
    }

    location /media/ {
        alias /home/ubuntu/betopia-daily-server/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/daily.sock;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/daily /etc/nginx/sites-enabled/
```

---

## Step 7 — Fix Permissions (prevents 403 on static/media)

```bash
sudo chmod -R 755 /home/ubuntu/betopia-daily-server/staticfiles
sudo chmod o+rx /home/ubuntu /home/ubuntu/betopia-daily-server
```

---

## Step 8 — Start Everything

```bash
sudo systemctl daemon-reload

sudo systemctl enable daily.socket
sudo systemctl start daily.socket

sudo systemctl enable daily.service
sudo systemctl start daily.service

sudo nginx -t
sudo systemctl restart nginx
```

---

## Step 9 — SSL Certificate (HTTPS)

Point DNS first — add an A record for `server` pointing to your VPS IP in your domain registrar.

Then run:

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d server.betopiadaily.shop
```

Test auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

---

# PART 2 — SQLite to PostgreSQL Migration (without data loss)

---

## Step 1 — Install psycopg2

```bash
source env/bin/activate
pip install psycopg2-binary
```

---

## Step 2 — Export SQLite Data

```bash
python manage.py dumpdata --natural-foreign --natural-primary \
  --exclude=contenttypes --exclude=auth.permission \
  --indent 2 > datadump.json
```

Verify it's not empty:

```bash
ls -lh datadump.json
```

---

## Step 3 — Create PostgreSQL Database & User

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE betopia_daily;
CREATE USER betopia_daily_user WITH PASSWORD 'yourpassword';
ALTER ROLE betopia_daily_user SET client_encoding TO 'utf8';
ALTER ROLE betopia_daily_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE betopia_daily_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE betopia_daily TO betopia_daily_user;
GRANT ALL ON SCHEMA public TO betopia_daily_user;
ALTER DATABASE betopia_daily OWNER TO betopia_daily_user;
\q
```

> ⚠️ The last two lines are required on PostgreSQL 15+ — without them Django cannot create tables.

---

## Step 4 — Update Django Settings (switch to PostgreSQL)

In `config/settings.py`, replace the `DATABASES` block:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'betopia_daily',
        'USER': 'betopia_daily_user',
        'PASSWORD': 'yourpassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

---

## Step 5 — Run Migrations on PostgreSQL

```bash
python manage.py migrate --run-syncdb
```

---

## Step 6 — Load Data into PostgreSQL

```bash
python manage.py loaddata datadump.json
```

You should see:
```
Installed X object(s) from 1 fixture(s)
```

If it fails, try:
```bash
python manage.py loaddata datadump.json --ignorenonexistent
```

---

## Step 7 — Delete datadump.json

```bash
rm ~/betopia-daily-server/datadump.json
```

---

## Step 8 — Restart Gunicorn

```bash
sudo systemctl restart daily.service
sudo systemctl status daily.service
```

---

---

# PART 3 — Ongoing Operations

---

## Restart Services (after code changes)

```bash
sudo systemctl restart daily.socket
sudo systemctl restart daily.service
sudo systemctl restart nginx
```

---

## Debug Commands

```bash
# Gunicorn status
sudo systemctl status daily.service
sudo systemctl status daily.socket

# Check socket file exists
file /run/daily.sock

# Nginx
sudo nginx -t
sudo tail -f /var/log/nginx/error.log

# Gunicorn logs
sudo journalctl -u daily.service -n 50
```

---

## Test URLs

```
https://server.betopiadaily.shop/admin/
https://server.betopiadaily.shop/static/admin/css/base.css   ← confirm CSS loads
https://server.betopiadaily.shop/media/
```

---

## Full Request Flow

```
Browser
   ↓
Nginx (port 443 HTTPS)
   ↓
/static → ~/betopia-daily-server/staticfiles/
/media  → ~/betopia-daily-server/media/
/       → /run/daily.sock
   ↓
Gunicorn
   ↓
Django (config.wsgi)
   ↓
PostgreSQL (betopia_daily)
```

---

## Notes

- Each Django project on this VPS needs its own socket, service, and nginx config
- `.socket` = systemd configuration file
- `.sock` = runtime Unix socket file created at startup
- Nginx serves static/media directly for performance (never hits Gunicorn)
- PostgreSQL 15+ requires explicit `GRANT ALL ON SCHEMA public` for new users