# 🌐 Reverse SSH Tunnel Setup (Raspberry Pi → VPS) — **Final Polished Version**

## 🎯 Goal

Access a private Raspberry Pi (behind NAT) from anywhere using a public VPS.

```
Client → VPS (public_ip:PORT) → Raspberry Pi (ssh:22)
```

---

# 🧱 Prerequisites

* Public VPS (Linux, SSH enabled)
* Raspberry Pi (private network)
* SSH access on both systems
* Firewall (UFW or similar) on VPS

---

# 🛠️ STEP 1 — VPS Configuration

## 1.1 Create dedicated SSH key (tunnel-only)

```bash
mkdir -p ~/.ssh/tunnel_keys
ssh-keygen -t ed25519 -f ~/.ssh/tunnel_keys/pi_tunnel -N "" -C "reverse tunnel key"
```

---

## 1.2 Restrict key (🔥 IMPORTANT HARDENING)

Edit:

```bash
nano ~/.ssh/authorized_keys
```

Add:

```bash
no-agent-forwarding,no-X11-forwarding,no-pty,permitopen="localhost:22" ssh-ed25519 AAAA... reverse tunnel key
```

👉 Replace `AAAA...`

### 🔐 What this does:

* ❌ No shell access
* ❌ No interactive login
* ✅ Only allows port forwarding → `localhost:22`

---

## ⚠️ OPTIONAL (only if using `*:` or `0.0.0.0:`)

Edit:

```bash
sudo nano /etc/ssh/sshd_config
```

Add:

```bash
GatewayPorts clientspecified
AllowTcpForwarding yes
```

Then:

```bash
sudo systemctl restart ssh
```

---

## 1.3 Allow port on firewall

```bash
sudo ufw allow 5522/tcp
```

---

# 🍓 STEP 2 — Raspberry Pi Configuration

## 2.1 Copy private key

```bash
scp user@<VPS_IP>:~/.ssh/tunnel_keys/pi_tunnel ~/.ssh/
chmod 600 ~/.ssh/pi_tunnel
```

⚠️ Remove from VPS:

```bash
rm ~/.ssh/tunnel_keys/pi_tunnel
```

---

## 2.2 Test tunnel manually (FIRST TRUTH TEST)

```bash
ssh -i ~/.ssh/pi_tunnel -N -R 5522:localhost:22 user@<VPS_IP>
```

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

From VPS:

```bash
ssh -p 5522 pi_user@localhost
```

---

## 🌍 External Access (optional)

If you want public access:

```bash
ssh -i ~/.ssh/pi_tunnel -N -R 0.0.0.0:5522:localhost:22 user@<VPS_IP>
```

Then:

```bash
ssh -p 5522 pi_user@<VPS_IP>
```

---

# 🔄 STEP 4 — Persistent Tunnel (autossh)

## 4.1 Install

```bash
sudo apt update
sudo apt install autossh -y
```

---

## 4.2 Create systemd service (🔥 FIXED VERSION)

```bash
sudo nano /etc/systemd/system/reverse-tunnel.service
```

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

## 🔥 Why this version matters

* ❌ Removed broken multiline (`\`)
* ✅ Reliable reconnect behavior
* ✅ Fails fast if tunnel not created
* ✅ Works with systemd properly

---

## 4.3 Enable

```bash
sudo systemctl daemon-reload
sudo systemctl enable reverse-tunnel
sudo systemctl restart reverse-tunnel
```

---

## 4.4 Debug

```bash
journalctl -u reverse-tunnel -f
```

---

# 🔐 STEP 5 — Fail2Ban (VPS Protection)

## Install

```bash
sudo apt install fail2ban -y
```

---

## Config

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

## Start

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## Check

```bash
sudo fail2ban-client status
```

---

# 🧠 Security Summary (CRITICAL)

### MUST DO:

* ✅ SSH keys only
* ✅ Disable password login:

```bash
sudo nano /etc/ssh/sshd_config
```

```ini
PasswordAuthentication no
```

---

* ✅ Restart SSH:

```bash
sudo systemctl restart ssh
```

---

* ✅ Use high random port (e.g. 5522, 49222)
* ✅ Enable Fail2Ban
* ✅ Restrict tunnel key

---

# 🏁 Final Result

You now have:

* 🌍 Public access to private Pi
* 🔁 Auto-reconnecting tunnel
* 🔐 Locked-down SSH
* 🛡️ Brute-force protection
