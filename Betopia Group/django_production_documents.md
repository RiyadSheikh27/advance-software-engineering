```md
# Betopia VPS — Django (Gunicorn + Nginx + Systemd Socket) Deployment

## Project

| App | Folder | Virtual Env | Socket | Domain |
|---|---|---|---|---|
| Django Backend (limited) | `~/betopia-limited-server` | `~/betopia-limited-server/env` | `/run/limited.sock` | dashboard.betopialimited.com |

---

## Project Structure

```

betopia-limited-server/
│
├── apps/
├── config/               # Django project (settings, wsgi)
├── staticfiles/          # collectstatic output
├── media/                # uploaded files
├── env/                  # virtual environment
├── manage.py
└── db.sqlite3

````

---

## Systemd Files

| Type | File |
|---|---|
| Socket | `/etc/systemd/system/limited.socket` |
| Service | `/etc/systemd/system/limited.service` |

### Commands

```bash
# edit socket
sudo nano /etc/systemd/system/limited.socket

# edit service
sudo nano /etc/systemd/system/limited.service

# reload systemd
sudo systemctl daemon-reload
````

---

## Systemd Socket (limited.socket)

```ini
[Unit]
Description=betopia limited socket

[Socket]
ListenStream=/run/limited.sock

[Install]
WantedBy=sockets.target
```

---

## Systemd Service (limited.service)

```ini
[Unit]
Description=betopia limited gunicorn service
Requires=limited.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/betopia-limited-server

ExecStart=/home/ubuntu/betopia-limited-server/env/bin/gunicorn \
    --workers 3 \
    --access-logfile - \
    --bind unix:/run/limited.sock \
    config.wsgi:application

[Install]
WantedBy=multi-user.target
```

---

## Nginx Config

| App        | Config File                          |
| ---------- | ------------------------------------ |
| Django App | `/etc/nginx/sites-available/limited` |

### Enable Site

```bash
sudo ln -s /etc/nginx/sites-available/limited /etc/nginx/sites-enabled/
```

---

## Nginx Server Block

```nginx
server {
    listen 80;
    server_name dashboard.betopialimited.com www.dashboard.betopialimited.com;

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location /static/ {
        alias /home/ubuntu/betopia-limited-server/staticfiles/;
        autoindex off;
        expires 30d;
        add_header Cache-Control "public";
    }

    location /media/ {
        alias /home/ubuntu/betopia-limited-server/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/limited.sock;
    }
}
```

---

## Django Settings (Important)

```python
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

ALLOWED_HOSTS = [
    "dashboard.betopialimited.com",
    "www.dashboard.betopialimited.com"
]
```

---

## Collect Static

```bash
source env/bin/activate
python manage.py collectstatic
```

---

## Permissions Fix (Fix 403 errors)

```bash
sudo chmod -R 755 /home/ubuntu/betopia-limited-server/staticfiles
sudo chmod o+rx /home/ubuntu /home/ubuntu/betopia-limited-server
```

---

## System Startup Commands

```bash
sudo systemctl daemon-reload

sudo systemctl enable limited.socket
sudo systemctl start limited.socket

sudo systemctl enable limited.service
sudo systemctl start limited.service
```

---

## Restart Services

```bash
sudo systemctl restart limited.socket
sudo systemctl restart limited.service
sudo systemctl restart nginx
```

---

## Debug Commands

```bash
sudo systemctl status limited.service
sudo systemctl status limited.socket

file /run/limited.sock

sudo nginx -t

sudo tail -f /var/log/nginx/error.log
```

---

## Testing URLs

```
http://dashboard.betopialimited.com/static/admin/css/base.css
http://dashboard.betopialimited.com/media/
```

---

## Full Request Flow

```
Browser
   ↓
Nginx
   ↓
/static → staticfiles/
/media  → media/
/      → gunicorn.sock
   ↓
Gunicorn
   ↓
Django (config.wsgi)
```

---

## Notes

* Each Django project needs its own:

  * socket
  * service
  * nginx config

* `.socket` = systemd configuration

* `.sock` = runtime communication file

* Nginx serves static/media directly for performance

```

---

If you want next upgrade, I can also:
- :contentReference[oaicite:0]{index=0}
- or :contentReference[oaicite:1]{index=1}
```
