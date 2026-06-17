# Betopia VPS — Next.js Servers

## Projects

| App | Folder | Port | Domain |
|---|---|---|---|
| Frontend (client) | `~/betopia-group-client` | 6001 | betopiagroup.com |
| Dashboard (backend) | `~/betopia-group-dashboard` | 6005 | dashboard.betopiagroup.com |
| Daily (frontend) | `~/Betopia-Daily` | 3000 | betopiadaily.shop |

---

## Nginx Config Files

| App | Config File |
|---|---|
| Frontend | `/etc/nginx/sites-available/betopia` |
| Dashboard | `/etc/nginx/sites-available/betopia-dashboard` |
| Daily | `/etc/nginx/sites-available/betopia-daily` |

```bash
# Open configs
sudo nano /etc/nginx/sites-available/betopia
sudo nano /etc/nginx/sites-available/betopia-dashboard
sudo nano /etc/nginx/sites-available/betopia-daily

# List all enabled sites
ls /etc/nginx/sites-enabled/

# List all available sites
ls /etc/nginx/sites-available/
```

---

## SSL Certificates

| Domain | Certificate Path |
|---|---|
| betopiagroup.com | `/etc/letsencrypt/live/betopiagroup.com/` |
| dashboard.betopiagroup.com | `/etc/letsencrypt/live/dashboard.betopiagroup.com/` |
| betopiadaily.shop | `/etc/letsencrypt/live/betopiadaily.shop/` |

```bash
# List all certificates
sudo certbot certificates

# Renew
sudo certbot renew --dry-run
```

---

## PM2 Processes

| PM2 Name | App | Folder |
|---|---|---|
| `betopia` | Frontend client | `~/betopia-group-client` |
| `betopia-dashboard` | Dashboard | `~/betopia-group-dashboard` |
| `betopia-daily` | Daily frontend | `~/Betopia-Daily` |

```bash
# View all running apps
pm2 status

# View logs
pm2 logs betopia
pm2 logs betopia-dashboard
pm2 logs betopia-daily
```

---

## Restarting Apps

```bash
# Restart individually
pm2 restart betopia
pm2 restart betopia-dashboard
pm2 restart betopia-daily

# Restart all at once
pm2 restart all
```

---

## Nginx

```bash
# Test config
sudo nginx -t

# Graceful reload (preferred)
sudo systemctl reload nginx

# Full restart
sudo systemctl restart nginx
```

> Nginx reloads all sites at once — there's no per-site reload. But only the file you edited will have changed, so it's safe to reload globally.

---

## After Code Changes

```bash
# ── Group Client ───────────────────────────────────────────
cd ~/betopia-group-client
git pull
npm run build
pm2 restart betopia

# ── Group Dashboard ────────────────────────────────────────
cd ~/betopia-group-dashboard
git pull
npm run build
pm2 restart betopia-dashboard

# ── Daily ──────────────────────────────────────────────────
cd ~/Betopia-Daily
git pull
npm run build
pm2 restart betopia-daily
```

---

## After VPS Reboot

PM2 will auto-start all apps. If they don't come up:

```bash
pm2 resurrect
```

Verify everything is up:

```bash
pm2 status
sudo systemctl status nginx
```

Full reboot:

```bash
sudo reboot
```

---

## Environment Files

```bash
cat ~/betopia-group-client/.env
cat ~/betopia-group-dashboard/.env
cat ~/Betopia-Daily/.env
```