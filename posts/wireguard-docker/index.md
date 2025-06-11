# How to Run a WireGuard Tunnel Inside a Docker Container Without Kernel Support


# How to Run a WireGuard Tunnel Inside a Docker Container Without Kernel Support

If your NAS or Linux host does not support the WireGuard kernel module — like the Synology DS220+ — you can still run a secure WireGuard tunnel using Docker and `wireguard-go`. This article walks you through the full setup with downloadable examples and scripts.

---

## 🫠 Why This Setup?

Many embedded Linux systems (e.g., Synology NAS, LXC, OpenVZ VPS) lack support for kernel modules like `wireguard.ko`. Fortunately, the Go-based `wireguard-go` project allows running WireGuard entirely in user space. When combined with Docker, it enables a portable and rootless VPN tunnel.

---

## 🔧 What You'll Achieve

* A working WireGuard VPN tunnel inside a Docker container
* Fully functional without kernel support
* Persistent setup for backup, replication, or remote access
* Ready-to-use configuration and generation scripts

---

## 📚 Requirements

* Synology DS220+ (or similar Linux NAS)
* Docker and SSH/root access
* `x86_64` architecture (for the provided binary)

---

## 📁 Folder Structure

```
wg-peer/
├── Dockerfile
├── docker-compose.yml
├── entrypoint.sh
├── wireguard-go        # Precompiled user-space binary
├── wg0.conf            # WireGuard peer config
├── genkeys.sh          # Key + config generator
├── install.sh          # Curl-based installer
├── watchdog_wg.sh      # Optional cron watchdog
├── README.md
```

---

## 🚀 Quick Setup

Before starting the tunnel, follow these steps to prepare the environment.

### ✅ 1. Download Files

Use the install script to download all necessary files:

```bash
curl -fsSL https://run.topli.ch/docker/wg-peer/install.sh | bash
cd wg-peer
```

---

### 🔐 2. Generate WireGuard Keys

Run the key generation script:

```bash
./genkeys.sh
```

This will:

* Generate a `PrivateKey` and `PublicKey` using a temporary container
* Save a template `wg0.conf` file in the current directory

---

### ✍️ 3. Edit `wg0.conf`

Open the generated file and **update the following**:

* `PublicKey` of your WireGuard **server**
* `Endpoint` — IP or domain of the server (e.g. `vpn.example.com:51820`)
* Optional: update `Address` or `AllowedIPs` to match your network

---

### 🌐 4. Ensure Network Access

Make sure:

* UDP port `51820` (or the one used) is **open** on your server
* The system supports `--privileged` Docker containers
* Host network mode is available (`network_mode: host`)

---

### ▶️ 5. Start the Tunnel

Now you can build and run the container:

```bash
docker-compose up -d --build
```

To check if the tunnel is working:

```bash
docker exec -it wg-peer wg show
```

---

## 🛀 Example wg0.conf

```ini
[Interface]
PrivateKey = <your_private_key>
Address = 10.10.1.6/32

[Peer]
PublicKey = <server_public_key>
Endpoint = <your-server-ip>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

---

## 🚚 Launch Container Manually

If not using `docker-compose`, run manually:

```bash
docker build -t wg-peer .
docker run -d --name wg-peer --privileged --network host wg-peer
```

Use `--privileged` and `--network host` to allow full tunnel capabilities.

---

## 🧾 View Logs and Debug

To view logs from the running container:

```bash
docker logs wg-peer
```

To enter the container's shell (for inspection or testing):

```bash
docker exec -it wg-peer bash
```

From inside the container, you can also run:

```bash
wg show
```

to check tunnel status directly.

---

## ♻️ Clean Shutdown & Restart

```bash
docker stop wg-peer
docker start wg-peer
```

---

## 🛡 Watchdog via Cron (optional)

Monitor tunnel health and auto-restart if down:

```bash
crontab -e
```

Add:

```cron
*/5 * * * * /path/to/wg-peer/watchdog_wg.sh >> /var/log/wg-watchdog.log 2>&1
```

---

## 🔧 Manually Compile wireguard-go (if needed)

```bash
apt install -y golang git

git clone https://git.zx2c4.com/wireguard-go
cd wireguard-go
go build
cp wireguard /usr/bin/wireguard-go
chmod +x /usr/bin/wireguard-go
```

---

## 🔗 Useful Links

* 🔐 [WireGuard Documentation](https://www.wireguard.com/)
* 📂 [GitHub Source Files](https://github.com/toplich/run/tree/main/docker/wg-peer)
* 📅 [Install Script](https://run.topli.ch/docker/wg-peer/install.sh)
* 🔐 [Full README](https://run.topli.ch/docker/wg-peer/README.md)

---

️⃣ Need help with customization or scaling? Reach out via [blog.topli.ch](https://blog.topli.ch/) or GitHub.

© 2025 run.topli.ch


