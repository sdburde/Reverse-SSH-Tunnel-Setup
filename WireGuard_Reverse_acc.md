# 🌐 Reverse Access using WireGuard (Pi ↔ VPS)

Now we upgrade from a rope bridge to a **private underground network** 🕳️⚡
No exposed ports, no fragile tunnels — just a clean private LAN over the internet.

---

## 🎯 Goal

Instead of exposing ports, we create a **secure private network**:

```text
Laptop → VPS (public) ⇄ WireGuard ⇄ Raspberry Pi (private)
```

👉 Then access Pi directly:

```bash
ssh pi_user@10.0.0.2
```

---

# 🧠 Why WireGuard > autossh

| Feature      | autossh 🔁 | WireGuard ⚡  |
| ------------ | ---------- | ------------ |
| Persistent   | ✅          | ✅ (native)   |
| Multi-device | ❌ messy    | ✅ scalable   |
| Performance  | medium     | 🔥 very fast |
| Security     | good       | 🔐 excellent |
| Maintenance  | manual     | minimal      |

---

# 🛠️ STEP 1 — Install WireGuard

## On BOTH VPS and Pi:

```bash
sudo apt update
sudo apt install wireguard -y
```

👉 If install fails:

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install wireguard -y
```

---

# 🔑 STEP 2 — Generate keys (VERY IMPORTANT)

## On VPS:

```bash
wg genkey | tee ~/server.key | wg pubkey > ~/server.pub
```

## On Pi:

```bash
wg genkey | tee ~/pi.key | wg pubkey > ~/pi.pub
```

---

## 🔍 View keys (THIS is what “PASTE” means)

On Pi:

```bash
cat ~/pi.key     # use in Pi config
cat ~/pi.pub     # use in VPS config
```

On VPS:

```bash
cat ~/server.key # use in VPS config
cat ~/server.pub # use in Pi config
```

---

## ⚠️ Key Rules (CRITICAL)

* PrivateKey = **secret (never share)**
* PublicKey = **shared**
* Keys must be:

  * single line
  * ~44 chars
  * no quotes, no spaces

---

# 🧱 STEP 3 — Configure VPS (Server)

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <PASTE server.key>

# Enable IP forwarding
PostUp = sysctl -w net.ipv4.ip_forward=1
PostDown = sysctl -w net.ipv4.ip_forward=0

[Peer]
PublicKey = <PASTE pi.pub>
AllowedIPs = 10.0.0.2/32
```

---

# 🍓 STEP 4 — Configure Raspberry Pi (Client)

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <PASTE pi.key>

[Peer]
PublicKey = <PASTE server.pub>
Endpoint = <VPS_IP>:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```

---

## 💡 Why `PersistentKeepalive = 25`

Keeps NAT open so Pi stays reachable
Without it → tunnel randomly dies 💀

---

# 🔥 STEP 5 — Firewall (CRITICAL)

## 5.1 VPS firewall (UFW)

```bash
sudo ufw allow 51820/udp
sudo ufw reload
```

---

## 🚨 5.2 Azure Network Security Group (VERY IMPORTANT 👀)

Azure has an **extra firewall layer (NSG)** that blocks traffic even if UFW allows it.

👉 Go to:

**Azure Portal → VM → Networking → Inbound rules → Add rule**

| Field    | Value                           |
| -------- | ------------------------------- |
| Port     | 51820                           |
| Protocol | UDP                             |
| Action   | Allow                           |
| Priority | 1000 (or any free lower number) |

---

## 🔍 Verify VPS is listening

```bash
sudo ss -ulnp | grep 51820
```

Expected:

```text
udp   UNCONN   0   0   0.0.0.0:51820
```

---

# 🔐 STEP 6 — Fix permissions (IMPORTANT)

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

---

# 🚀 STEP 7 — Start WireGuard

## First test manually (recommended)

```bash
sudo wg-quick up wg0
```

👉 If error → fix before systemd

---

## If already running (important fix)

```bash
sudo wg-quick down wg0
```

---

## Then enable service

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

---

# 🧪 STEP 8 — Verify tunnel

## Check status:

```bash
sudo wg
```

---

## ✅ Expected (WORKING)

```text
latest handshake: X seconds ago
transfer: X received, Y sent
endpoint: <ip>:port
```

---

## ❌ If you see:

```text
latest handshake: (never)
transfer: 0 B received
```

👉 Means:

* Firewall issue (UFW / Azure NSG)
* Wrong endpoint IP

---

## Ping test

```bash
ping 10.0.0.1   # from Pi
ping 10.0.0.2   # from VPS
```

---

# 🎯 STEP 9 — SSH (final result)

```bash
ssh pi_user@10.0.0.2
```

---

# 🌍 Access Pi from your laptop

## Option 1 — via VPS

```bash
ssh -J user@<VPS_IP> pi_user@10.0.0.2
```

---

## Option 2 — connect laptop to WireGuard (best)

```bash
ssh pi_user@10.0.0.2
```

---

# 🛡️ Security advantages

* ❌ No open SSH port on Pi
* ❌ No reverse tunnel exposure
* 🔐 End-to-end encryption
* ✅ Only trusted peers connect

---

# 🧠 Mental model

* autossh → *“Pi calls VPS”* ☎️
* WireGuard → *“Pi and VPS share same private LAN”* 🏠

---

# ⚠️ Common errors (real-world)

### ❌ “Key is not correct length”

→ wrong key / malformed key

---

### ❌ wg0 already exists

→ fix:

```bash
sudo wg-quick down wg0
```

---

### ❌ No handshake (MOST COMMON)

→ fix:

* open UFW port
* open Azure NSG port
* verify endpoint IP

---

### ❌ Transfer shows only sent, no received

→ firewall blocking UDP

---

# ⚡ When to use what

| Use case              | Choose    |
| --------------------- | --------- |
| Quick hack            | autossh   |
| Stable infra          | WireGuard |
| Multi-device          | WireGuard |
| CV pipelines (you 👀) | WireGuard |

---

# 🔥 Pro insight (for your CV/ML work)

This unlocks:

* edge devices (Pi cameras)
* distributed inference
* private data pipelines
* remote GPU orchestration

You basically built your own **mini cloud network** ☁️⚡

---

# 🏁 Final Result

You now have:

* 🌐 Private network (Pi + VPS)
* 🔐 Secure encrypted tunnel
* ⚡ Direct SSH without ports
* 🧠 Scalable architecture
