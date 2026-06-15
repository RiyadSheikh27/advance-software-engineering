# Betopia VPS — Hosted Projects & Files

## Projects

| App | Folder | Port | Domain |
|---|---|---|---|
| Frontend (client) | `~/betopia-group-client` | 6001 | betopiagroup.com |
| Dashboard (backend) | `~/betopia-group-dashboard` | 6005 | dashboard.betopiagroup.com |

---

## Nginx Config Files

| App | Config File |
|---|---|
| Frontend | `/etc/nginx/sites-available/betopia` |
| Dashboard | `/etc/nginx/sites-available/betopia-dashboard` |

### Commands to access

```bash
# View frontend nginx config
sudo cat /etc/nginx/sites-available/betopia

# View dashboard nginx config
sudo cat /etc/nginx/sites-available/betopia-dashboard

# List all enabled sites
ls /etc/nginx/sites-enabled/

# List all available sites
ls /etc/nginx/sites-available/
```

---

## SSL Certificates

```bash
# List all certificates
sudo certbot certificates
```

| Domain | Certificate Path |
|---|---|
| betopiagroup.com | `/etc/letsencrypt/live/betopiagroup.com/` |
| dashboard.betopiagroup.com | `/etc/letsencrypt/live/dashboard.betopiagroup.com/` |

---

## PM2 Processes

```bash
# View all running apps
pm2 status

# View logs
pm2 logs betopia
pm2 logs betopia-dashboard
```

| PM2 Name | App |
|---|---|
| `betopia` | Frontend client |
| `betopia-dashboard` | Dashboard |

---

## Restarting the Server

### Restart both apps
```bash
pm2 restart betopia
pm2 restart betopia-dashboard
```

### Restart Nginx
```bash
sudo systemctl reload nginx   # graceful reload (preferred)
sudo systemctl restart nginx  # full restart
```

### Reload a specific site's Nginx config
```bash
# Edit the config first
sudo nano /etc/nginx/sites-available/betopia
# or
sudo nano /etc/nginx/sites-available/betopia-dashboard

# Then test and reload
sudo nginx -t && sudo systemctl reload nginx
```
> Nginx reloads all sites at once — there's no per-site reload. But only the file you edited will have changed, so it's safe to reload globally.

### After a VPS reboot
PM2 will auto-start both apps. If for any reason they don't come up:
```bash
pm2 resurrect
```

### Full server reboot
```bash
sudo reboot
```
After reboot, SSH back in and verify everything is up:
```bash
pm2 status
sudo systemctl status nginx
```

---

## Environment Files

```bash
# View env vars
cat ~/betopia-group-client/.env
cat ~/betopia-group-dashboard/.env
```