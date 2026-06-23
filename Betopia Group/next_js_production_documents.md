# Betopia VPS — Next.js Servers

## Projects

| App | Folder | Port | Domain |
|---|---|---|---|
| Frontend (client) | `~/betopia-group-client` | 6001 | betopiagroup.com |
| Dashboard (backend) | `~/betopia-group-dashboard` | 6005 | dashboard.betopiagroup.com |
| Daily (frontend) | `~/Betopia-Daily` | 3000 | betopiadaily.shop |
| PulseGrid (frontend) | `~/PulseGrid-Betopia` | 3007 | betopiapulsegrid.com |

---

## Nginx Config Files

| App | Config File |
|---|---|
| Frontend | `/etc/nginx/sites-available/betopia` |
| Dashboard | `/etc/nginx/sites-available/betopia-dashboard` |
| Daily | `/etc/nginx/sites-available/betopia-daily` |
| PulseGrid | `/etc/nginx/sites-available/pulse-grid` |

```bash
# Open configs
sudo nano /etc/nginx/sites-available/betopia
sudo nano /etc/nginx/sites-available/betopia-dashboard
sudo nano /etc/nginx/sites-available/betopia-daily
sudo nano /etc/nginx/sites-available/pulse-grid

# List all enabled sites
ls /etc/nginx/sites-enabled/

# List all available sites
ls /etc/nginx/sites-available/
```

> ⚠️ **Always reload after creating/editing/symlinking a config.** Writing or symlinking a file into `sites-enabled/` does **not** apply it — Nginx only re-reads `sites-enabled/` on reload/restart. Forgetting this caused PulseGrid to silently serve Betopia Daily's content for a while (see Troubleshooting Log below). Always run:
> ```bash
> sudo nginx -t && sudo systemctl reload nginx
> ```
> immediately after touching any Nginx config.

---

## SSL Certificates

| Domain | Certificate Path |
|---|---|
| betopiagroup.com | `/etc/letsencrypt/live/betopiagroup.com/` |
| dashboard.betopiagroup.com | `/etc/letsencrypt/live/dashboard.betopiagroup.com/` |
| betopiadaily.shop | `/etc/letsencrypt/live/betopiadaily.shop/` |
| betopiapulsegrid.com | `/etc/letsencrypt/live/betopiapulsegrid.com/` |

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
| `pulse-grid` | PulseGrid frontend | `~/PulseGrid-Betopia` |

```bash
# View all running apps
pm2 status

# View logs
pm2 logs betopia
pm2 logs betopia-dashboard
pm2 logs betopia-daily
pm2 logs pulse-grid
```

> **Note on starting `pulse-grid`:** It does not use a fixed port inside `package.json`'s start script (`"start": "next start"`), so the port must be passed explicitly both via env var and the `-p` flag, or it will default to 3000 and collide with `betopia-daily`:
> ```bash
> PORT=3007 pm2 start npm --name "pulse-grid" -- start -- -p 3007
> ```

---

## Restarting Apps

```bash
# Restart individually
pm2 restart betopia
pm2 restart betopia-dashboard
pm2 restart betopia-daily
pm2 restart pulse-grid

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
>
> `sudo nginx -T` dumps the config **on disk**, not what the running worker currently has loaded — it can look correct even if a reload is still pending. When in doubt, just reload.

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

# ── PulseGrid ──────────────────────────────────────────────
cd ~/PulseGrid-Betopia
git pull
rm -rf .next
npm install
npm run build
pm2 restart pulse-grid
```

> `pm2 restart` alone re-runs the existing build — it does **not** rebuild. Always run `npm run build` (and for PulseGrid, clear `.next` first) before restarting, or PM2 will keep serving stale content.

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
cat ~/PulseGrid-Betopia/.env
```

---

## Troubleshooting Log

### PulseGrid serving Betopia Daily's content (June 2026)
**Symptom:** `betopiapulsegrid.com` loaded Betopia Daily's site instead of PulseGrid's.

**Causes found, in order (three separate bugs stacked together):**
1. **Port collision** — `pulse-grid` had no fixed port in its start script, defaulted to 3000, which was already taken by `betopia-daily`. Fixed by always starting with `PORT=3007 ... -p 3007`.
2. **Orphaned process** — an old `next-server` child process from a previous `betopia-daily` restart was still holding port 3000 outside of PM2's tracking (`pwdx <pid>` revealed it). Fixed with `sudo kill -9 <pid>`.
3. **Stale build** — `pulse-grid`'s `.next` folder was outdated/corrupted. Fixed with `rm -rf .next node_modules && npm install && npm run build`.
4. **Root cause (the actual fix):** the `pulse-grid` Nginx config was correctly written and symlinked into `sites-enabled/`, but **Nginx was never reloaded** after the symlink was created. The running Nginx worker had no route for `betopiapulsegrid.com` loaded in memory, so requests fell through to the `betopia-daily` server block. `sudo nginx -T` looked correct (it reads from disk) — only an actual reload fixed it:
   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

**Lesson:** Any new site config — always `nginx -t && systemctl reload nginx` right after symlinking it into `sites-enabled/`. Don't assume "config file looks right" means "Nginx is using it."