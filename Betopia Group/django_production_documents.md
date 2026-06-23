# Betopia VPS — Django Servers

## Projects

| App                      | Folder                      | Virtual Env                     | Socket              | Domain                       |
| ------------------------ | --------------------------- | ------------------------------- | ------------------- | ---------------------------- |
| Django Backend (limited) | `~/betopia-limited-server`  | `~/betopia-limited-server/env`  | `/run/limited.sock` | dashboard.betopialimited.com |
| Django Backend (daily)   | `~/betopia-daily-server`    | `~/betopia-daily-server/env`    | `/run/daily.sock`   | server.betopiadaily.shop     |
| CPMO Project Management  | `~/CPMO-Project-Management` | `~/CPMO-Project-Management/env` | `/run/cpmo.sock`    | pm-cpmo.betopiagroup.com     |

---

## Key Files

| Type             | Limited                                       | Daily                                       | CPMO                                           |
| ---------------- | --------------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| Django Settings  | `~/betopia-limited-server/config/settings.py` | `~/betopia-daily-server/config/settings.py` | `~/CPMO-Project-Management/config/settings.py` |
| Gunicorn Socket  | `/etc/systemd/system/limited.socket`          | `/etc/systemd/system/daily.socket`          | `/etc/systemd/system/cpmo.socket`              |
| Gunicorn Service | `/etc/systemd/system/limited.service`         | `/etc/systemd/system/daily.service`         | `/etc/systemd/system/cpmo.service`             |
| Nginx Config     | `/etc/nginx/sites-available/limited`          | `/etc/nginx/sites-available/daily`          | `/etc/nginx/sites-available/cpmo`              |

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

# ── CPMO ───────────────────────────────────────────────────
nano ~/CPMO-Project-Management/config/settings.py
sudo nano /etc/systemd/system/cpmo.socket
sudo nano /etc/systemd/system/cpmo.service
sudo nano /etc/nginx/sites-available/cpmo
```

---

## Database

| App     | Engine     | Database        | User                 |
| ------- | ---------- | --------------- | -------------------- |
| Limited | SQLite     | `db.sqlite3`    | —                    |
| Daily   | PostgreSQL | `betopia_daily` | `betopia_daily_user` |
| CPMO    | SQLite     | `db.sqlite3`    | —                    |

```bash
# Access PostgreSQL (daily)
sudo -u postgres psql
\c betopia_daily
\dt
\q
```

---

## SSL Certificates

| Domain                       | Certificate Path                                      |
| ---------------------------- | ----------------------------------------------------- |
| dashboard.betopialimited.com | `/etc/letsencrypt/live/dashboard.betopialimited.com/` |
| server.betopiadaily.shop     | `/etc/letsencrypt/live/server.betopiadaily.shop/`     |
| pm-cpmo.betopiagroup.com     | `/etc/letsencrypt/live/pm-cpmo.betopiagroup.com/`     |

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

sudo systemctl status cpmo.socket
sudo systemctl status cpmo.service

# ── Restart (after code changes) ───────────────────────────
sudo systemctl restart limited.socket
sudo systemctl restart limited.service

sudo systemctl restart daily.socket
sudo systemctl restart daily.service

sudo systemctl restart cpmo.socket
sudo systemctl restart cpmo.service

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

# ── CPMO ───────────────────────────────────────────────────
source ~/CPMO-Project-Management/env/bin/activate
cd ~/CPMO-Project-Management
python manage.py collectstatic
sudo chmod -R 755 /home/ubuntu/CPMO-Project-Management/staticfiles
sudo chmod o+rx /home/ubuntu /home/ubuntu/CPMO-Project-Management
```

---

## After VPS Reboot

Both services are enabled on startup. If they don't come up:

```bash
sudo systemctl start limited.socket
sudo systemctl start limited.service

sudo systemctl start daily.socket
sudo systemctl start daily.service

sudo systemctl start cpmo.socket
sudo systemctl start cpmo.service

sudo systemctl restart nginx
```

Verify:

```bash
sudo systemctl status limited.service
sudo systemctl status daily.service
sudo systemctl status cpmo.service
sudo systemctl status nginx
```

---

## Debug

```bash
# ── Gunicorn logs ──────────────────────────────────────────
sudo journalctl -u limited.service -n 50
sudo journalctl -u daily.service -n 50
sudo journalctl -u cpmo.service -n 50

# ── Nginx error log ────────────────────────────────────────
sudo tail -f /var/log/nginx/error.log

# ── Check socket files exist ───────────────────────────────
file /run/limited.sock
file /run/daily.sock
file /run/cpmo.sock
```

---

## Test URLs

```text
# Limited
https://dashboard.betopialimited.com/admin/
https://dashboard.betopialimited.com/static/admin/css/base.css

# Daily
https://server.betopiadaily.shop/admin/
https://server.betopiadaily.shop/static/admin/css/base.css

# CPMO
https://pm-cpmo.betopiagroup.com/admin/
https://pm-cpmo.betopiagroup.com/static/admin/css/base.css
```

---

## Full Request Flow

```text
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

## CPMO Deployment Notes

```text
Project Path:
~/CPMO-Project-Management

Socket:
 /run/cpmo.sock

Nginx Config:
 /etc/nginx/sites-available/cpmo

Gunicorn Service:
 /etc/systemd/system/cpmo.service

Gunicorn Socket:
 /etc/systemd/system/cpmo.socket

Domain:
 pm-cpmo.betopiagroup.com
```

---

## Notes

* Each Django project needs its own socket, service, and nginx config.
* `.socket` = systemd configuration file.
* `.sock` = runtime Unix socket file created at startup.
* Nginx serves static/media directly — never hits Gunicorn.
* PostgreSQL 15+ requires `GRANT ALL ON SCHEMA public` for new users.
* CPMO admin CSS is served through `/static/` mapped to `~/CPMO-Project-Management/staticfiles/`.
* Run `collectstatic` whenever static assets change.
