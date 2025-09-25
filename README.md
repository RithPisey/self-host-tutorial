# Step-by-step VPS hardening tutorial

Nice — below is a compact, practical checklist and commands you can run **now** to lock down a brand-new Ubuntu VPS. I’ve kept it ordered so you can follow one step at a time. Read each step before you run commands and **keep a fallback session** (or console access from your VPS provider) open while changing SSH settings so you don’t accidentally lock yourself out.

---

## 1) Connect and update the system

1. SSH into your VPS:

```bash
ssh root@YOUR_SERVER_IP
```

2. Refresh package lists and upgrade everything:

```bash
sudo apt update
sudo apt upgrade -y
```

3. If kernel updated, check if reboot required:

```bash
[ -f /var/run/reboot-required ] && echo "REBOOT REQUIRED"
```

If required, reboot (use provider console if you prefer):

```bash
sudo reboot
```

---

## 2) Create a non-root user + give sudo

1. Add a regular user (replace `alice`):

```bash
sudo adduser alice
```

2. Add that user to the `sudo` group:

```bash
sudo usermod -aG sudo alice
```

3. Verify groups:

```bash
groups alice
id alice
```

4. Test by logging in as that user from another terminal:

```bash
ssh alice@YOUR_SERVER_IP
# then try:
sudo whoami   # should print 'root' after you enter alice's password
```

---

## 3) Generate SSH keypair (on *your local machine*) and install the public key

On your **local** computer (not the VPS):

```bash
# generate modern ED25519 key
ssh-keygen -t ed25519 -C "your_email@example.com"
# default locations (~/.ssh/id_ed25519 and id_ed25519.pub)
```

Copy your public key to the VPS (recommended):

```bash
# either:
ssh-copy-id alice@YOUR_SERVER_IP

# or manual (if ssh-copy-id unavailable)
cat ~/.ssh/id_ed25519.pub | ssh alice@YOUR_SERVER_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

Quick check: log out and back in; password should not be required:

```bash
ssh alice@YOUR_SERVER_IP
```

---

## 4) Disable SSH password authentication and root SSH login (DON’T do this until your SSH key test works)

**Important:** keep an active root or provider console session until you confirm key login works for your non-root user.

Edit SSH daemon config:

```bash
sudo nano /etc/ssh/sshd_config
```

Make these changes (uncomment / set accordingly):

```
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
# (Optional) To change SSH port: Port 2222
```

Save and exit. Then restart SSH:

```bash
sudo systemctl restart sshd
```

Test from another terminal first (do not close working session yet):

```bash
ssh alice@YOUR_SERVER_IP     # should succeed using key
ssh root@YOUR_SERVER_IP      # should be denied
```

If you changed the port, remember to specify it: `ssh -p 2222 alice@IP`.

---

## 5) Firewall (UFW) basics — block everything except what you need

1. Set safe defaults, allow SSH and enable UFW:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH     # or sudo ufw allow 2222/tcp if you changed SSH port
sudo ufw enable
sudo ufw status verbose
```

2. If you need a webserver later:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

3. To restrict SSH to a single IP (if you have a static IP):

```bash
sudo ufw delete allow OpenSSH
sudo ufw allow from 203.0.113.45 to any port 22 proto tcp
```

(Replace `203.0.113.45` with your IP and port with your SSH port.)

---

## 6) Automatic security updates

Install and enable unattended upgrades:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

You can tweak `/etc/apt/apt.conf.d/50unattended-upgrades` to control which packages are auto-updated and whether the server auto-reboots:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

---

## 7) Add extra protections (recommended)

* **Fail2ban** — blocks IPs that fail authentication repeatedly:

```bash
sudo apt install fail2ban -y
# basic service will work; edit /etc/fail2ban/jail.local to customize
```

* **Disable unused services**: list listening ports and stop services you don't need:

```bash
sudo ss -tuln     # shows listening TCP/UDP ports
sudo systemctl disable --now SERVICE_NAME
```

* **Install and use a reverse proxy / TLS** later (nginx/caddy) for web apps.

---

## 8) Hardening checklist & safety notes

* ✅ Confirm key login for `alice` **before** disabling password auth.
* ✅ Do **not** close your active root/provider console session until SSH changes verified.
* ✅ Keep at least one sudo user with key access.
* ✅ Monitor logs: `sudo tail -f /var/log/auth.log`
* ✅ Consider using 2FA for control panels, and keep backups of critical configs and keys.
* ✅ Regularly run `sudo apt update && sudo apt upgrade -y`.

---

## Useful commands summary

```bash
# system updates
sudo apt update && sudo apt upgrade -y

# create user and give sudo
sudo adduser alice
sudo usermod -aG sudo alice

# ssh key copy (from local machine)
ssh-copy-id alice@SERVER_IP

# check open ports
sudo ss -tuln

# enable ufw firewall
sudo ufw default deny incoming
sudo ufw allow OpenSSH
sudo ufw enable

# install fail2ban
sudo apt install fail2ban -y

# unattended upgrades
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades

# restart sshd
sudo systemctl restart sshd
```

---

## Final notes

* Changing the SSH port or disabling password login reduces *automated* brute-force attempts drastically — but **always** verify connectivity first.
* Keep a provider console (VPS web UI) access method available in case of mistakes.
* Next steps you might want: set up `fail2ban` rules, configure `nginx`/`caddy` with TLS, or run services in Docker with limited privileges. If you want, I can generate a short script to apply the safe defaults above (with prompts), or produce a walkthrough video script based on this tutorial.

Want that script or an automated checklist file (bash)?
