# 🌐 Reverse SSH Tunnel Setup (Raspberry Pi → VPS)

## 🎯 Goal

Access a private Raspberry Pi (behind NAT) from anywhere using a public VPS.

```
Client → VPS (public_ip:PORT) → Raspberry Pi (ssh:22)
```

---

# 🧱 Prerequisites

* Public VPS (Linux, SSH enabled)
* Raspberry Pi (inside private network)
* SSH access on both systems
* UFW or firewall enabled on VPS

---

# 🛠️ STEP 1 — VPS Configuration

## 1.1 Create dedicated SSH key (for tunnel only)

```bash
mkdir -p ~/.ssh/tunnel_keys
ssh-keygen -t ed25519 -f ~/.ssh/tunnel_keys/pi_tunnel -N "" -C "reverse tunnel key"
```

---

## 1.2 Restrict key permissions

Edit:

```bash
nano ~/.ssh/authorized_keys
```

Add (single line):

```bash
no-agent-forwarding,no-X11-forwarding,no-pty,permitopen="localhost:22" ssh-ed25519 AAAA... reverse tunnel key
```

👉 Replace `AAAA...` with your `.pub` key

### 🔐 What this enforces:

* No shell access
* No interactive login
* Only allows forwarding → localhost:22

---

## 1.3 Allow external port

### 📌 Port Guidelines (important)

* You can use **any free TCP port**
* Recommended range: **20000–65000**
* Avoid common ports: `22, 80, 443, 8080`

```bash
sudo ufw allow 5522/tcp
```

---

## ⚠️ 1.4 Enable reverse tunnel public binding (ONLY if using `*:` or public access)

Edit:

```bash
sudo nano /etc/ssh/sshd_config
```

Add:

```bash
GatewayPorts clientspecified
AllowTcpForwarding yes
```

Restart:

```bash
sudo systemctl restart ssh
```

---

# 🍓 STEP 2 — Raspberry Pi Configuration

## 2.1 Copy private key from VPS

```bash
scp user@<VPS_IP>:~/.ssh/tunnel_keys/pi_tunnel ~/.ssh/
chmod 600 ~/.ssh/pi_tunnel
```

⚠️ Delete it from VPS after copying:

```bash
rm ~/.ssh/tunnel_keys/pi_tunnel
```

---

## 2.2 Test reverse tunnel manually (UPDATED ⚠️)

```bash
ssh -i ~/.ssh/pi_tunnel -N -R 5522:localhost:22 user@<VPS_IP>
```

👉 Removed `*:` (prevents GatewayPorts dependency issues)

---

## 2.3 Verify on VPS

```bash
ss -tlnp | grep 5522
```

Expected:

```
127.0.0.1:5522
```

---

# 🚀 STEP 3 — Access Raspberry Pi

### From VPS:

```bash
ssh -p 5522 pi_user@localhost
```

---

### 🌍 From external machine (ONLY if using `*:` or `0.0.0.0:`)

```bash
ssh -p 5522 pi_user@<VPS_IP>
```

---

# 🔄 STEP 4 — Persistent Tunnel (autossh) on the Raspberry Pi 🍓

## 📌 Important

* Runs ONLY on Raspberry Pi
* SSH must work **without password** before this step

---

## 4.1 Install autossh

```bash
sudo apt update
sudo apt install autossh -y
```

---

## 4.2 Create systemd service (FIXED ⚠️)

```bash
sudo nano /etc/systemd/system/reverse-tunnel.service
```

Paste:

```ini
[Unit]
Description=Persistent Reverse SSH Tunnel
After=network-online.target
Wants=network-online.target

[Service]
User=pi_user
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -N -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -i /home/pi_user/.ssh/pi_tunnel -R 5522:localhost:22 user@<VPS_IP>
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 🔥 Why this change matters

* ❌ Removed multiline `\` (systemd bug)
* ❌ Removed `*:` (avoids SSH config dependency)
* ✅ Ensures tunnel fails fast if broken
* ✅ More stable reconnect

---

## 4.3 Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable reverse-tunnel
sudo systemctl restart reverse-tunnel
```

---

## 🔍 Debug if needed

```bash
journalctl -u reverse-tunnel -f
```

---

# 🔐 STEP 5 — Fail2Ban (Intrusion Protection)

## 🛠️ Install Fail2Ban on VPS (FIXED ⚠️)

```bash
sudo apt update
sudo apt install fail2ban -y
```

👉 If error: *Unable to locate package*

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install fail2ban -y
```

---

## 📁 Basic configuration

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = 5522
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 3600
findtime = 600
```

---

## ▶️ Start Fail2Ban

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## 🔍 Check status

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

# 🧠 Security Summary (Critical)

### MUST DO:

* Use SSH keys only
* Disable password login:

```bash
sudo nano /etc/ssh/sshd_config
```

```ini
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
```

---

* Use high port (e.g. 5522)
* Enable Fail2Ban
* Restrict tunnel key

---

# 🏁 Final Result

You now have:

* 🌍 Public access to private Pi
* 🔁 Auto-reconnecting tunnel
* 🔐 Locked-down SSH
* 🛡️ Active intrusion prevention

