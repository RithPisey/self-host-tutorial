# Self-hosting step-by-step tutorial (Caddy, DNS, static sites, reverse proxy, PM2)

Below I converted your multi-part guide into a compact, actionable step-by-step tutorial you can follow. Copy & paste commands on the server (adjust usernames, domains, and IPs). I kept important notes and warnings inline so you don’t accidentally lock yourself out.

---

# Part 0 — Prerequisites & quick checklist

* You have a VPS (Ubuntu/Debian recommended) and its public IP.
* You have a domain name and access to its DNS management (registrar or Cloudflare).
* You have a non-root local SSH user (we’ll use `cj` in examples).
* Firewall: make sure ports **80** and **443** will be allowed (UFW or provider firewall).
* Important: if using Cloudflare, **disable the orange proxy** for any host Caddy should provision certificates for (grey cloud).

---

# Part 1 — DNS (map names → your server IP)

1. Log into your registrar or Cloudflare.
2. Create A records pointing to your server IP (set **TTL low** while testing — e.g. 60s):

   * `@` → `YOUR.SERVER.IP` (root / apex domain)
   * `www` → `YOUR.SERVER.IP`
   * `holidays` → `YOUR.SERVER.IP` (example subdomain)
3. **If using Cloudflare**: set proxy to **off** (grey cloud). Caddy needs direct connections for TLS issuance.
4. Verify with `dig` from your terminal:

```bash
dig +short notlocalhost.net
dig +short www.notlocalhost.net
dig +short holidays.notlocalhost.net
```

(wait TTL seconds after creating records if needed)

---

# Part 2 — Install Caddy as a system service (automatic HTTPS)

1. Install Caddy (Debian/Ubuntu):

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

2. Verify the service:

```bash
sudo service caddy status
caddy version
```

3. Create a sites organization for maintainability:

```bash
sudo mkdir -p /etc/caddy/sites-available /etc/caddy/sites-enabled
sudo chown root:root /etc/caddy/sites-available /etc/caddy/sites-enabled
```

4. Edit main Caddyfile to import enabled sites:

```bash
sudo nano /etc/caddy/Caddyfile
```

Replace/ensure it contains:

```caddy
# /etc/caddy/Caddyfile
notlocalhost.net {
    root * /usr/share/caddy
    file_server
}

import sites-enabled/*
```

5. Prefer **reload** (zero-downtime) when applying changes:

```bash
sudo service caddy reload
```

(You can use `restart` while testing, but `reload` is recommended once running multiple sites.)

---

# Part 3 — Redirects: force HTTPS & canonical domain

1. Create a redirect site file:

```bash
sudo nano /etc/caddy/sites-available/http-redirects
```

Contents:

```caddy
:80, www.notlocalhost.net {
    # redirect any HTTP or www request to canonical HTTPS domain
    redir https://notlocalhost.net{uri}
}
```

2. Enable it:

```bash
sudo ln -s /etc/caddy/sites-available/http-redirects /etc/caddy/sites-enabled/http-redirects
sudo service caddy reload
```

---

# Part 4 — Serve a static website (fileserver)

1. Create directories and set ownership (replace user `cj`):

```bash
sudo mkdir -p /var/www/notlocalhost.net
sudo chown cj:cj /var/www/notlocalhost.net
sudo chmod 755 /var/www/notlocalhost.net
```

2. Update the main Caddy site block to point at the new root:

```caddy
# in /etc/caddy/Caddyfile
notlocalhost.net {
    root * /var/www/notlocalhost.net
    file_server
    # SPA support (if needed): try_files {path} {path}/ /index.html
}
```

3. Transfer site files (from local machine) — examples:

* SCP:

```bash
scp -r ./site/* cj@notlocalhost.net:/var/www/notlocalhost.net/
```

* rsync (efficient incremental sync):

```bash
rsync -avzP ./site/ cj@notlocalhost.net:/var/www/notlocalhost.net/
```

* Git (public repo):

```bash
# on server as cj in an empty target directory
git clone https://github.com/you/repo.git /var/www/notlocalhost.net
```

4. Prevent exposing `.git` or other sensitive folders — use `hide` in file_server:

```caddy
notlocalhost.net {
    root * /var/www/notlocalhost.net
    file_server {
        hide .git
    }
    handle_errors {
        respond "{status} {status_text}"
    }
}
```

5. Reload Caddy:

```bash
sudo service caddy reload
```

---

# Part 5 — Reverse proxy a dynamic app (example: Node.js API)

## A — Open ports

Make sure ports 80/443 allowed (UFW example):

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

## B — Install Node.js (NVM) as non-root user

Run as the non-root user (cj):

```bash
# install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc

# install latest LTS
nvm install --lts
node -v
npm -v
```

## C — Clone app (private repo: use deploy key; public repo: HTTPS)

* **Public repo** example (as `cj`):

```bash
mkdir -p ~/hosts
git clone https://github.com/you/holidays.git ~/hosts/holidays
cd ~/hosts/holidays
npm install
# copy .env from example and set values
cp .env.sample .env
# start locally for test
npm start
```

* **Private repo (recommended)**: create a **deploy key**:

  1. On the **server** (cj user), generate an SSH key pair with a unique name:

  ```bash
  ssh-keygen -t ed25519 -C "deploy@notlocalhost" -f ~/.ssh/deploy_holidays -N ""
  ```

  2. Copy the public key `~/.ssh/deploy_holidays.pub` and add it to the GitHub repo under **Settings → Deploy keys** (read-only).
  3. Clone via SSH:

  ```bash
  GIT_SSH_COMMAND='ssh -i ~/.ssh/deploy_holidays' git clone git@github.com:you/holidays.git ~/hosts/holidays
  ```

## D — Configure Caddy reverse proxy for the subdomain

Create `/etc/caddy/sites-available/holidays-api`:

```caddy
holidays.notlocalhost.net {
    reverse_proxy localhost:5000
    handle_errors {
        respond "{status} {status_text}"
    }
    log {
        output file /var/log/caddy/holidays.access.log
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/caddy/sites-available/holidays-api /etc/caddy/sites-enabled/holidays-api
sudo mkdir -p /var/log/caddy
sudo chown caddy:caddy /var/log/caddy
sudo service caddy reload
```

---

# Part 6 — Keep the app running: PM2 (recommended for Node)

1. Install PM2 (as `cj`):

```bash
npm install -g pm2
```

2. Start the app with PM2 (from project folder):

```bash
cd ~/hosts/holidays
pm2 start npm --name "holidays-api" -- start     # or pm2 start dist/index.js --name holidays-api
pm2 list
```

3. Enable startup on reboot (PM2→systemd):

```bash
pm2 startup
# follow printed command (run it with sudo)
pm2 save
```

4. Manage logs:

```bash
pm2 logs holidays-api
pm2 stop holidays-api
pm2 restart holidays-api
pm2 delete holidays-api
```

---

# Part 7 — Build pipelines / apps that require a build step

* For TypeScript/compiled backends: `npm run build` → run the built `dist` files with PM2.
* For frontend frameworks (Vite, CRA): run `npm run build` → serve the generated `dist`/`build` folder with the static file server at `/var/www/...`.

Example: build & serve

```bash
# in project root (on server or CI)
npm install
npm run build

# copy/build output to /var/www/site
rsync -avz build/ cj@notlocalhost.net:/var/www/bigtimer/
# configure caddy root to /var/www/bigtimer
```

---

# Part 8 — Logging & error handling (Caddy snippets)

* Add structured access logs and simple error responses in site blocks:

```caddy
log {
    output file /var/log/caddy/access.log {
        roll_size 50mb
        roll_keep 5
        roll_keep_for 720h
    }
}

handle_errors {
    respond "{status} {status_text}"
}
```

(Place these inside the relevant site blocks or in included files.)

---

# Part 9 — Good practices & security notes

* **Disable Cloudflare proxy** for any domain Caddy will manage TLS for.
* Always **verify SSH key login** for your non-root user before disabling password auth or root login.
* Keep filesystem perms tight:

  * Site files: `owner=cj`, `group=cj`, `chmod 755` for dirs and `644` for files.
  * Don’t make site folders writable by the webserver user.
* Use `sudo service caddy reload` to apply config changes with no downtime.
* Use `unattended-upgrades` and `fail2ban` to reduce attack surface.
* Use deploy keys for private repos (one deploy key per repo) instead of adding the VPS key to your main GitHub account.
* Monitor `sudo tail -f /var/log/auth.log`, `pm2 logs`, and Caddy logs in `/var/log/caddy`.

---

# Quick command summary (copy-paste)

```bash
# Caddy install (Ubuntu)
sudo apt update
# (follow Caddy install block above)

# Create site folders
sudo mkdir -p /var/www/notlocalhost.net
sudo chown cj:cj /var/www/notlocalhost.net
sudo chmod 755 /var/www/notlocalhost.net

# Enable a site
sudo ln -s /etc/caddy/sites-available/holidays-api /etc/caddy/sites-enabled/holidays-api
sudo service caddy reload

# Install NVM & Node (as cj)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts

# PM2
npm install -g pm2
pm2 start npm --name "holidays-api" -- start
pm2 startup
pm2 save
```

---

