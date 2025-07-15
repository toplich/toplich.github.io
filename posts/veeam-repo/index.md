# Veeam Local Repositories in Practice: Hardened Linux, MinIO S3, iSCSI over WireGuard on Synology NAS


## 1. Direct-Attached Storage (DAS)

### Description:

DAS refers to a storage device (e.g., internal HDDs, USB drives, or RAID arrays) that is physically connected to the Veeam Backup Server or Backup Proxy.

### Use case:

Simple deployments or environments with limited infrastructure.

### Configuration Steps:

1. Connect the storage device to your Veeam server.
2. Format the disk using **ReFS** (Windows) or **XFS with reflink=1** (Linux) for best performance.
3. Open Veeam Console > Backup Infrastructure > Backup Repositories > Add Repository.
4. Choose "Direct Attached Storage" > Microsoft Windows or Linux (depending on the OS).
5. Specify the mount path (e.g., `D:\Backups` or `/mnt/veeam`).
6. Finish the wizard and assign the repository to a job.

---

## 2. NAS (SMB or NFS Share)

### Description:

Network-attached storage accessed over the network via SMB or NFS protocol.

### Use case:

Shared backup targets, easy scalability, centralized storage.

### Configuration Steps:

1. Ensure your NAS share is accessible (e.g., `\\nas01\veeam` or NFS path).
2. In Veeam Console > Add Backup Repository > Shared Folder.
3. Specify the network path and access credentials.
4. Assign a folder name and complete the wizard.

> ⚠️ Immutability and fast clone are **not** supported with SMB/NFS.

---

## 3. Windows Backup Repository

### Description:

A Windows Server or desktop system where Veeam stores backups. Best used with ReFS file system.

### Use case:

Windows-only environments or when ReFS benefits (fast clone) are desired.

### Configuration Steps:

1. Ensure the volume is formatted with ReFS (Windows Server 2016+):

   ```powershell
   Format-Volume -DriveLetter D -FileSystem ReFS -NewFileSystemLabel "VeeamRepo"
   ```
2. In Veeam Console > Add Backup Repository > Microsoft Windows.
3. Provide the hostname or IP and credentials.
4. Select the path (e.g., `D:\Backups`).
5. Finish the setup.

---

## 4. Hardened Linux Repository (Immutability Enabled)

### Description:

A secure, immutable Linux-based repository using non-root access and XFS with reflink.

### Use case:

Maximum protection against ransomware. Required for compliance-focused environments.

### Configuration Steps:

1. Prepare a Linux server with XFS-formatted disk and create a non-root user (e.g., `veeamrepo`).
2. Assign the public SSH key to the user (password login should remain enabled only temporarily for initial setup).
3. Temporarily grant `sudo` access for Veeam to install its data mover.
4. In Veeam Console, add the Linux server as a repository and enable immutability.
5. Once the repository is added successfully, **remove the user from the sudo group** and optionally create `/etc/sudoers.d/99-veeamrepo` with:

   ```bash
   veeamrepo ALL=(ALL) !ALL
   ```
6. Disable SSH password authentication:

   ```bash
   sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
   systemctl restart ssh
   ```
7. Final security hardening (optional):

   ```bash
   usermod -s /usr/sbin/nologin veeamrepo
   ```

### Manual Preparation Steps Summary

If you'd rather prepare the Hardened Linux Repository manually, follow these steps:

1. **Create a non-root user** with no password:

   ```bash
   adduser --disabled-password --gecos "" veeamrepo
   ```

2. **Assign a public SSH key**:

   ```bash
   mkdir -p /home/veeamrepo/.ssh
   echo "ssh-rsa AAAA..." > /home/veeamrepo/.ssh/authorized_keys
   chmod 700 /home/veeamrepo/.ssh
   chmod 600 /home/veeamrepo/.ssh/authorized_keys
   chown -R veeamrepo:veeamrepo /home/veeamrepo/.ssh
   ```

3. **Install required packages**:

   ```bash
   apt update && apt install -y xfsprogs openssh-server sudo
   ```

4. **Format and mount the XFS disk with reflink**:

   ```bash
   mkfs.xfs -b size=4096 -m reflink=1,crc=1 /dev/sdb
   mkdir -p /mnt/veeam
   UUID=$(blkid -s UUID -o value /dev/sdb)
   echo "UUID=$UUID /mnt/veeam xfs defaults 0 0" >> /etc/fstab
   mount -a
   chown veeamrepo:veeamrepo /mnt/veeam
   ```

5. **Add the repository to Veeam** using SSH key and temporary sudo access.

6. **After successful configuration** (optional hardening):

   ```bash
   deluser veeamrepo sudo
   echo "veeamrepo ALL=(ALL) !ALL" > /etc/sudoers.d/99-veeamrepo
   sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
   systemctl restart ssh
   usermod -s /usr/sbin/nologin veeamrepo
   ```

### Automated Script (hosted at [run.topli.ch](https://run.topli.ch))

If you'd prefer an automated solution, you can use the following script:

```bash
curl -fsSL https://run.topli.ch/sh/hardened-linux-repo.sh | bash
```

> ⚠️ Review the script before executing. It configures XFS, creates a non-root user, and prepares the system for Veeam Hardened Repository integration.

---

## 5. S3-Compatible Repository with MinIO (Docker on Synology NAS)

### Description:

A local object storage solution using MinIO with S3-compatible API, TLS encryption, object versioning, and Object Lock for immutability. Ideal for on-prem, private cloud setups with full control.

### Use case:

Secure backup target with immutability, suitable for use with Veeam's Object Storage Repository feature.

### Configuration Summary:

1. **Prepare Synology NAS:**

   * Ensure Docker (Container Manager) is installed
   * Enable SSH access

2. **Generate TLS Certificates:**

   ```bash
   mkdir certs
   openssl req -x509 -nodes -days 730 -newkey rsa:2048 \
     -keyout certs/private.key \
     -out certs/public.crt \
     -subj "/CN=minio.local" \
     -addext "subjectAltName=DNS:minio.local,DNS:localhost,IP:192.168.1.2,IP:127.0.0.1"
   chmod 600 certs/private.key
   chmod 644 certs/public.crt
   ```

3. **Create MinIO data directory:**

   ```bash
   mkdir -p /volume1/minio-data
   ```

4. **Deploy MinIO via Docker Compose:**
   Create `docker-compose.yml`:

   ```yaml
   version: '3.8'
   services:
     minio:
       image: minio/minio:latest
       container_name: minio
       ports:
         - "9000:9000"
         - "9001:9001"
       volumes:
         - /volume1/minio-data:/data
         - ./certs:/root/.minio/certs
       environment:
         MINIO_ROOT_USER: minio
         MINIO_ROOT_PASSWORD: yourSecretKey
         MINIO_SERVER_URL: https://192.168.1.2:9000
         MINIO_BROWSER_REDIRECT_URL: https://minio.local:9001
       command: server /data --console-address ":9001"
       restart: unless-stopped
   ```

   Then run:

   ```bash
   docker-compose up -d
   ```

5. **Create S3 bucket with Object Lock:**

   ```bash
   docker run --rm -it \
     -e AWS_ACCESS_KEY_ID=minio \
     -e AWS_SECRET_ACCESS_KEY=yourSecretKey \
     --network host \
     amazon/aws-cli \
     s3api create-bucket \
     --bucket veeam-locked \
     --object-lock-enabled-for-bucket \
     --endpoint-url https://192.168.1.2:9000 \
     --no-verify-ssl \
     --no-cli-pager
   ```

6. **Enable versioning and retention:**

   ```bash
   docker exec minio mc alias set local https://192.168.1.2:9000 minio yourSecretKey --insecure
   docker exec minio mc version enable local/veeam-locked --insecure
   docker exec minio mc retention set --default GOVERNANCE 30d local/veeam-locked --insecure
   ```

7. **Add to Veeam:**

   * Go to **Backup Infrastructure > Object Storage Repository**
   * Select **S3-Compatible**
   * Enter:

     * Endpoint: `https://192.168.1.2:9000`
     * Bucket: `veeam-locked`
     * Access Key: `minio`
     * Secret Key: `yourSecretKey`
   * Enable **Immutability**

> 🔗 Optional: Use this MinIO S3 target as Capacity Tier in a Scale-Out Backup Repository (SOBR).

---

## 6. iSCSI-Based Repository over WireGuard Tunnel
 Using Veeam Backup Copy via iSCSI Target on Synology NAS over WireGuard Tunnel

In this guide, we'll show how to configure a secure and stable Backup Copy destination in Veeam by connecting to an iSCSI target hosted on a Synology NAS, with traffic routed through a WireGuard tunnel. This setup is ideal for inter-site backup replication, where the NAS is behind NAT or has a dynamic IP address.

---

### 🔧 Use Case

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

### 🔎 Overview

* 📁 Synology NAS with iSCSI Target enabled (LUN created)
* 🛠 Windows Server with Veeam Backup & Replication
* 🔺 WireGuard tunnel between the two sites
* 🔗 iSCSI traffic routed securely via VPN tunnel
* 🌐 ReFS-formatted volume used as repository

---

### 📂 Step 1: Prepare Synology NAS

1. Install **SAN Manager** (if not already installed)
2. Create a **LUN** and **iSCSI Target**
3. Enable CHAP authentication (optional but recommended)
4. Ensure LUN is **mapped to the target**
5. Note internal NAS IP (e.g. `192.168.1.2`)

---

### 🛂 Step 2: Set Up WireGuard Tunnel

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

### 🛠 Step 3: Connect Windows to iSCSI Target

1. Open **iSCSI Initiator** on the Windows Server
2. Enter the \*\* IP of Synology\*\* (e.g. `192.168.1.2`)
3. Connect and enter CHAP credentials if configured
4. Once connected, open **Disk Management**
5. Initialize the disk and format with **ReFS**
6. Assign a static drive letter (e.g. `R:`)

---

### 📄 Step 4: Add Repository in Veeam

1. Open Veeam B\&R Console
2. Go to **Backup Infrastructure > Backup Repositories**
3. Add New Repository:

   * Type: **Direct Attached Storage**
   * Server: This Windows server
   * Path: `R:\`
   * Format: **ReFS** (optimized for fast cloning)

Use the repository as a target for **Backup Copy Jobs**.

---

### 📊 Performance & Reliability Tips

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

### 🔗 Summary

This setup allows using a Synology NAS as a reliable Backup Copy target for Veeam over a secure WireGuard VPN.  
It leverages iSCSI to expose a block-level device, giving better performance and integrity than SMB/NFS, especially over WAN.  
The use of ReFS on the repository ensures fast cloning and enhanced integrity, while automated watchdog mechanisms help keep both the VPN tunnel and iSCSI session continuously available.  
Together, these elements make this architecture a production-ready, fault-tolerant remote backup solution for hybrid environments.

🔐 [WireGuard](https://www.wireguard.com/)  |  📄 [Veeam Documentation](https://helpcenter.veeam.com/)

---

## 7. Immutability & Compatibility Matrix

| Repository Type                         | File System | Fast Clone | Immutability | TLS Support | Remote-Ready |
|----------------------------------------|-------------|------------|--------------|--------------|--------------|
| Direct-Attached (Windows)              | ReFS        | ✅         | ❌           | ❌           | ❌           |
| Direct-Attached (Linux)                | XFS         | ✅         | ❌           | ❌           | ❌           |
| Hardened Linux Repository              | XFS         | ✅         | ✅           | ❌           | ⚠️ (manual)  |
| NAS (SMB/NFS)                          | ext4/btrfs  | ❌         | ❌           | ❌           | ✅           |
| S3-Compatible with MinIO               | ObjectStore | ❌         | ✅           | ✅           | ✅           |
| iSCSI Target via WireGuard (ReFS)      | ReFS        | ✅         | ❌           | ✅ (via tunnel) | ✅         |

## 8. Feature Comparison Table

| Feature                      | Hardened Linux | SMB/NFS | Windows ReFS | MinIO S3 | iSCSI via WG |
|------------------------------|----------------|---------|---------------|----------|---------------|
| Fast Clone                  | ✅              | ❌      | ✅            | ❌       | ✅             |
| Immutability                | ✅              | ❌      | ❌            | ✅       | ❌             |
| TLS Encryption              | ❌              | ❌      | ❌            | ✅       | ✅             |
| Requires VPN                | ❌              | ❌      | ❌            | ❌       | ✅             |
| Suitable for Backup Copy    | ⚠️              | ✅      | ✅            | ✅       | ✅             |

## 9. References and Links

- 🔐 [WireGuard Official](https://www.wireguard.com/)
- 📄 [Veeam Help Center](https://helpcenter.veeam.com/)
- 📦 [MinIO Documentation](https://min.io/docs/)
- 💾 [Synology iSCSI Target Setup](https://kb.synology.com/en-global/DSM/help/SANManager/target)
- 📜 [ReFS Overview – Microsoft](https://learn.microsoft.com/en-us/windows-server/storage/refs/refs-overview)
- 🔧 [Hardened Repository by Veeam](https://www.veeam.com/kb4134)

## 10. Final Thoughts

Choosing the right local repository depends on your needs:

- 🛡️ **For immutability & security**: Choose **Hardened Linux** or **MinIO with Object Lock**.
- 🪟 **For quick Windows setup**: Use **Windows ReFS repository**.
- 🌍 **For hybrid or inter-site setups**: Use **WireGuard + iSCSI** or **MinIO via S3-compatible endpoint**.

✅ Always:
- Monitor repository health.
- Plan for enough storage.
- Regularly test backup restores.
- Secure credentials (CHAP, SSH keys).
- Use job retry and validation mechanisms in Veeam.

With the right architecture, even small environments can achieve enterprise-grade backup resilience.



