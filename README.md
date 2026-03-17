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

```bash
sudo ufw allow 5522/tcp
```

---

## 1.4 Enable reverse tunnel public binding

Edit:

```bash
sudo nano /etc/ssh/sshd_config
```

Add:

```bash
GatewayPorts clientspecified
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

## 2.2 Test reverse tunnel manually

```bash
ssh -i ~/.ssh/pi_tunnel -N -R *:5522:localhost:22 user@<VPS_IP>
```

---

## 2.3 Verify on VPS

```bash
ss -tlnp | grep 5522
```

Expected:

```
0.0.0.0:5522
```

---

# 🚀 STEP 3 — Access Raspberry Pi

From any external machine:

```bash
ssh -p 5522 pi_user@<VPS_IP>
```

---

# 🔄 STEP 4 — Persistent Tunnel (autossh) on the Raspberry Pi 🍓

## 4.1 Install autossh

```bash
sudo apt update
sudo apt install autossh -y
```

---

## 4.2 Create systemd service

```bash
sudo nano /etc/systemd/system/reverse-tunnel.service
```

Paste:

```ini
[Unit]
Description=Persistent Reverse SSH Tunnel
After=network.target

[Service]
User=pi_user
ExecStart=/usr/bin/autossh -M 0 -N \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -i /home/pi_user/.ssh/pi_tunnel \
  -R *:5522:localhost:22 \
  user@<VPS_IP>

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 4.3 Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable reverse-tunnel
sudo systemctl start reverse-tunnel
```

---

# 🔐 STEP 5 — Fail2Ban (Intrusion Protection)

## 🧠 What is Fail2Ban?

Fail2Ban is like a vigilant nightclub guard:

* Watches logs (SSH login attempts)
* Detects repeated failures
* Bans IPs automatically using firewall rules

---

## ⚙️ How it works

1. Reads logs (e.g., `/var/log/auth.log`)
2. Matches patterns like:

   ```
   Failed password for invalid user
   ```
3. If failures exceed threshold → blocks IP

---

## 🛠️ Install Fail2Ban on VPS

```bash
sudo apt update
sudo apt install fail2ban -y
```

---

## 📁 Basic configuration

Create config:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add:

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

## 🧩 Meaning of settings

| Setting    | Meaning                    |
| ---------- | -------------------------- |
| `maxretry` | Attempts before ban        |
| `findtime` | Time window (seconds)      |
| `bantime`  | How long IP is blocked     |
| `port`     | Your SSH port (important!) |

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

## 🧪 Example behavior

If attacker tries:

```
password1 ❌
password2 ❌
password3 ❌
password4 ❌
password5 ❌
```

👉 IP gets banned automatically 🚫

---

# 🧠 Security Summary (Critical)

### MUST DO:

* Use SSH keys only (`PasswordAuthentication no`)
* Use non-standard port (like 5522)
* Enable Fail2Ban
* Restrict tunnel key

---

# 🏁 Final Result

You now have:

* 🌍 Public access to private Pi
* 🔁 Auto-reconnecting tunnel
* 🔐 Locked-down SSH
* 🛡️ Active intrusion prevention

---

# ⚡ Optional Upgrade (Stealth Mode)

Instead of exposing publicly:

```bash
-R 127.0.0.1:5522:localhost:22
```

Then connect via:

```bash
ssh -J user@<VPS_IP> pi_user@localhost
```

