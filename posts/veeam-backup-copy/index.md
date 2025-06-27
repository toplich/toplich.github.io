# Using Veeam Backup Copy via iSCSI Target on Synology NAS over WireGuard Tunnel


# Using Veeam Backup Copy via iSCSI Target on Synology NAS over WireGuard Tunnel

In this guide, we'll show how to configure a secure and stable Backup Copy destination in Veeam by connecting to an iSCSI target hosted on a Synology NAS, with traffic routed through a WireGuard tunnel. This setup is ideal for inter-site backup replication, where the NAS is behind NAT or has a dynamic IP address.

---

## 🔧 Use Case

```goat
┌──────────────────┐             WireGuard Tunnel            ┌──────────────────┐
│  Windows Server  │  ◄───────────────────────────────────►  │   Synology NAS   │
│ ┌──────────────┐ │                                         │ ┌──────────────┐ │
│ │  Veeam B&R   │ │                                         │ │  iSCSI LUN   │ │
│ └──────────────┘ │                                         │ └──────────────┘ │
│     ReFS R:\     │                                         │ IP: 192.168.1.2  │
└─────────┬────────┘                                         └─────────┬────────┘
          │                                                            │         
          ▼                                                            ▼         
PowerShell Watchdog                                             Watchdog Script  
 (check-iscsi.ps1)                                              wg-watchdog.sh   
```

You want to use your Synology NAS located at a remote site as a **Backup Copy repository** for Veeam. However, due to dynamic WAN IP and security constraints, you need to connect via **WireGuard VPN**. iSCSI will be used as the transport protocol, and the target will be mounted on a Windows Server with Veeam installed.

---

## 🔎 Overview

* 📁 Synology NAS with iSCSI Target enabled (LUN created)
* 🛠 Windows Server with Veeam Backup & Replication
* 🔺 WireGuard tunnel between the two sites
* 🔗 iSCSI traffic routed securely via VPN tunnel
* 🌐 ReFS-formatted volume used as repository

---

## 📂 Step 1: Prepare Synology NAS

1. Install **SAN Manager** (if not already installed)
2. Create a **LUN** and **iSCSI Target**
3. Enable CHAP authentication (optional but recommended)
4. Ensure LUN is **mapped to the target**
5. Note internal NAS IP (e.g. `192.168.1.2`)

---

## 🛂 Step 2: Set Up WireGuard Tunnel

📘 For a detailed setup on WireGuard tunnel configuration:

- 🔧 **NAS Side (Docker-based tunnel):**  
  👉 [Quick Setup: WireGuard inside Docker on Synology NAS](https://blog.topli.ch/posts/wireguard-docker/#-quick-setup)

- 🔧 **OPNsense Firewall Side (Site-to-Site VPN):**  
  👉 [WireGuard Site-to-Site VPN with OPNsense](https://blog.topli.ch/posts/opnsense/#21-wireguard-site-to-site-vpn-setup)

Ensure the following:

- The NAS IP (e.g. `192.168.1.2`) is reachable from the Windows Veeam server via the tunnel.  
- Proper **UDP port forwarding** is in place (e.g., port `51820` forwarded to the container).  
- `AllowedIPs` include the NAS LAN (e.g., `192.168.1.0/24`).  
- ✅ You can run the tunnel inside a Docker container using `wireguard-go`, especially useful if your NAS does not support kernel modules.

🔁 Tunnel & Connection Monitoring (Watchdog)

To ensure maximum reliability, especially across WAN links or dynamic networks, it's recommended to implement a watchdog mechanism on both ends of the tunnel:

🔄 On the NAS (Container)
A cron-based watchdog script periodically checks tunnel connectivity and restarts the container if necessary:

- 📜 Full guide:  
  👉 [WireGuard Docker Watchdog via Cron](https://blog.topli.ch/posts/wireguard-docker/#-watchdog-via-cron-optional)

- 💡 Example cron job:
  ```cron
  */5 * * * * /bin/bash /scripts/wg-watchdog.sh >> /var/log/wg-watchdog.log 2>&1
  ```

🧠 On the Windows Server (PowerShell Watchdog)
Use a lightweight PowerShell script to monitor the iSCSI session and automatically reconnect in case of disconnection. This helps mitigate edge cases where the WireGuard tunnel is restored, but the iSCSI session remains dropped.

📜 Script download:
👉 PowerShell Watchdog: [check-iscsi.ps1](run.topli.ch/ps/check-iscsi.ps1)

💡 You can schedule this script every 5 minutes via Task Scheduler or as a Windows Service:

- **Task Name:** `Check-iSCSI-Session`  
- **Trigger:** Every 5 minutes  
- **Action (command):**
  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File "C:\Scripts\check-iscsi.ps1"
  ```
---

## 🛠 Step 3: Connect Windows to iSCSI Target

1. Open **iSCSI Initiator** on the Windows Server
2. Enter the \*\* IP of Synology\*\* (e.g. `192.168.1.2`)
3. Connect and enter CHAP credentials if configured
4. Once connected, open **Disk Management**
5. Initialize the disk and format with **ReFS**
6. Assign a static drive letter (e.g. `R:`)

---

## 📄 Step 4: Add Repository in Veeam

1. Open Veeam B\&R Console
2. Go to **Backup Infrastructure > Backup Repositories**
3. Add New Repository:

   * Type: **Direct Attached Storage**
   * Server: This Windows server
   * Path: `R:\`
   * Format: **ReFS** (optimized for fast cloning)

Use the repository as a target for **Backup Copy Jobs**.

---

## 📊 Performance & Reliability Tips

* Use **WireGuard KeepAlive**: `PersistentKeepalive = 25`
* Enable **iSCSI MPIO** on Windows for redundancy (optional)
* Schedule cron to monitor the tunnel and reconnect if needed
* Use Veeam health checks and periodic validation

---

🧩 Additional Best Practices

✔️ Use hostnames instead of raw IPs
In iSCSI Initiator, configure target connections with DNS names (e.g., nas.remote.lan) to allow future migrations or failover.

✔️ Isolate iSCSI traffic inside VPN
Ensure no fallback to public routes by explicitly binding iSCSI to the WireGuard interface.

✔️ Avoid Backup Copy during maintenance
Run jobs outside known maintenance windows (e.g., NAS updates, WAN restarts).

✔️ Enable Veeam Job Retry
In job settings, allow retries (e.g., 3 times every 15 min) to auto-heal short tunnel drops.

✔️ Enable Veeam Integrity Checks
Activate health checks to validate backups and detect silent errors.

✔️ Secure CHAP credentials
Avoid storing CHAP secrets in plaintext — use Windows Credential Manager or secure vaults.

---

## 🔗 Summary

This setup allows using a Synology NAS as a reliable Backup Copy target for Veeam over a secure WireGuard VPN.  
It leverages iSCSI to expose a block-level device, giving better performance and integrity than SMB/NFS, especially over WAN.  
The use of ReFS on the repository ensures fast cloning and enhanced integrity, while automated watchdog mechanisms help keep both the VPN tunnel and iSCSI session continuously available.  
Together, these elements make this architecture a production-ready, fault-tolerant remote backup solution for hybrid environments.

---

🔗 [Blog: run.topli.ch](https://run.topli.ch/)  |  🔐 [WireGuard](https://www.wireguard.com/)  |  📄 [Veeam Documentation](https://helpcenter.veeam.com/)

© 2025 run.topli.ch


