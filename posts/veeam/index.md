# Veeam Backup: Best Practices and Configuration Guide


# 🔐 Veeam Backup: Best Practices and Configuration Guide

Veeam Backup & Replication is a powerful tool for protecting your data across virtual, physical, and cloud environments. This article focuses on key architectural decisions, **backup security best practices**, and real-world **deployment scenarios**.

---

## 🔐 Secure Backup with Immutability

{{% admonition type="info" title="Why This Matters" %}}
Immutability is the foundation of a secure backup strategy. It protects your backups from ransomware, human error, and malicious insiders by making data undeletable for a defined retention period.
{{% /admonition %}}

### 1.1 ✨ Use Immutable Storage

{{% details summary="▶ Recommended Linux Configuration" %}}

```bash
# Format disk with reflink support
sudo mkfs.xfs -m reflink=1 /dev/sdb
sudo mkdir -p /mnt/veeam
sudo mount /dev/sdb /mnt/veeam

# Create hardened backup user
sudo useradd -m -s /sbin/nologin veeamrepo
sudo chown veeamrepo: /mnt/veeam
```

* Use **XFS with reflink** for Veeam immutability
* Mount target to `/mnt/veeam`
* Configure in Veeam as "Hardened Repository"

{{% /details %}}

{{% admonition type="tip" title="Best Practice" %}}
Never use root for backup repositories. Harden access with non-root accounts and disable SSH password login.
{{% /admonition %}}

### 1.2 🔒 S3 Object Lock & Retention

{{% details summary="▶ Setup S3 Bucket with Object Lock (e.g., MinIO, AWS)" %}}

1. Create a bucket with `Object Lock` enabled
2. Enable `Versioning`
3. Set default **retention policy** (e.g., 30 days)

```bash
# Example with MinIO Client
mc alias set minio https://minio.example.com minioadmin minioadmin
mc retention set --default GOVERNANCE 30d minio/veeam --insecure
```

{{% /details %}}

{{% admonition type="tip" title="Security Tip" %}}
Assign **write-only access** for Veeam user to prevent accidental or malicious deletions.
{{% /admonition %}}

### 1.3 🔄 Apply the 3-2-1-1-0 Rule

{{% admonition type="info" title="Backup Strategy" %}}

* **3** copies of data (1 production + 2 backups)
* **2** different media types (e.g., disk + object storage)
* **1** offsite backup
* **1** immutable copy
* **0** errors after recovery verification (e.g., SureBackup)
  {{% /admonition %}}

### 1.4 🔐 Limit Access & Isolate Backup Networks

{{% details summary="▶ Network & Identity Hardening" %}}

* Use **separate credentials** for:

  * Backup console
  * Repositories
  * Hypervisors
  * S3 buckets
* Protect backup console with:

  * Multi-Factor Authentication (2FA)
  * Role-Based Access Control (RBAC)
* Isolate repository access with:

  * VLAN segmentation
  * Firewall rules
  * SSH key-only access

{{% /details %}}

{{% admonition type="danger" title="Do Not" %}}
Never expose repositories directly to the internet. Backup data should only be reachable via secured Veeam components.
{{% /admonition %}}

### 1.5 📊 Monitoring & Patching

{{% details summary="▶ Stay Ahead of Threats" %}}

* Enable **email notifications** for job results and warnings
* Regularly review **backup logs** and audit access
* Apply **Veeam updates and patches** promptly
* Integrate with monitoring platforms (e.g., Zabbix, Icinga)

{{% /details %}}

{{% admonition type="tip" title="Pro Tip" %}}
Schedule periodic restore tests and SureBackup jobs to validate recovery integrity.
{{% /admonition %}}

---

## ⚙️ Initial Setup of Veeam B\&R

{{% admonition type="info" title="Overview" %}}
Before configuring backup jobs, Veeam must be properly installed and its infrastructure components set up. This includes the main server, proxies, repositories, and connections to hypervisors.
{{% /admonition %}}

### 2.1 🖥 Install Veeam Backup & Replication

{{% details summary="▶ System Requirements" %}}

* **Operating System**: Windows Server 2016 or newer
* **SQL Server**: Use built-in SQL Express or an external instance
* **Hardware**:

  * 2+ vCPUs
  * 8+ GB RAM (16 GB recommended)
  * SSD for configuration and database preferred

{{% /details %}}

{{% details summary="▶ Installation Steps" %}}

1. Download Veeam B\&R ISO
2. Mount ISO and run `Setup.exe`
3. Choose **Backup & Replication**
4. Follow the wizard and install:

   * Veeam Backup Service
   * Console
   * Explorers
   * Transport & Catalog Services

{{% /details %}}

### 2.2 🧱 Add Backup Infrastructure

{{% details summary="▶ Add Hypervisors" %}}

* Go to `Backup Infrastructure > Managed Servers`
* Add:

  * **VMware vCenter Server**
  * or **Microsoft Hyper-V hosts**

Use service accounts with minimal required permissions.

{{% /details %}}

{{% details summary="▶ Add Backup Proxies" %}}

* Navigate to `Backup Infrastructure > Backup Proxies`
* Add Windows Server (VM or physical)
* Assign transport mode:

  * Direct SAN Access
  * Hot-Add (for VMware)
  * Network (NBD)

{{% /details %}}

{{% details summary="▶ Add Repositories" %}}

Supported types:

* ✅ Linux repo (XFS + reflink for immutability)
* ✅ Windows repo (ReFS for fast clone)
* ✅ NAS (SMB/NFS, no immutability)
* ✅ S3-compatible object storage

Use the appropriate wizard to configure credentials and paths.

{{% /details %}}

### 2.3 📦 Create Backup Jobs

{{% details summary="▶ Job Configuration Steps" %}}

1. Go to `Home > Jobs > Backup Job (VMs)`
2. Choose source: VMs, physical agents, or NAS
3. Select a **repository** (e.g., Hardened Linux, S3, iSCSI)
4. Set backup mode:

   * Incremental
   * Forever Forward Incremental
5. Define retention policy (e.g., 14 restore points)
6. (Optional) Enable:

   * Synthetic Full backups
   * Health Checks
   * Backup Copy

{{% /details %}}

{{% admonition type="tip" title="Best Practice" %}}
Assign each job to a repository that matches its role: fast storage for daily, immutable for long-term.
{{% /admonition %}}

### 2.4 ⚙️ Enable Optional Jobs

{{% details summary="▶ Optional Job Types" %}}

* **Backup Copy Job**

  * For offsite or secondary backup
* **SureBackup Job**

  * Automatically boots and verifies backups
* **Replication Job**

  * Creates hot standby VMs at DR location

{{% /details %}}

### 2.5 🔍 Monitor and Test

{{% details summary="▶ Recommended Monitoring" %}}

* Enable **email notifications**
* Check job logs and statistics regularly
* Integrate with external monitoring (Zabbix, Icinga)

{{% /details %}}

{{% details summary="▶ Restore Testing Options" %}}

* Full VM restore
* File-level restore
* Application-item restore (Exchange, SQL, etc.)
* Instant VM Recovery

Schedule test restores at least **once per month**.

{{% /details %}}

---

## 💾 Choosing the Right Repository

Choosing the correct repository type is fundamental to achieving the right balance of performance, cost-efficiency, immutability, and remote readiness.

{{% admonition type="info" title="Repository Types Explained" %}}
Veeam supports a variety of repository types that differ in performance, immutability, remote access, and compliance readiness.
{{% /admonition %}}

### 3.1 🗃️ Common Repository Types

- **ReFS (Windows)**  
  ✅ Fast clone support  
  ❌ No immutability  
  ⚠️ Needs VPN for remote usage

- **XFS (Linux)**  
  ✅ Reflink support  
  ✅ Immutability (when used in hardened mode)  
  ❌ No native remote access

- **Hardened Linux Repository**  
  🔐 Secure non-root access  
  ✅ Immutability with XFS  
  ❌ Requires SSH (not S3-compatible)

- **NAS (SMB/NFS)**  
  🟡 Easy to set up  
  ❌ No immutability  
  ✅ Can be remote (mountable)

- **S3-Compatible (MinIO, AWS, Wasabi)**  
  ✅ Object Lock  
  ✅ Versioning  
  ✅ TLS support  
  ✅ Ideal for long-term, remote, immutable storage

- **iSCSI over VPN (e.g., via WireGuard)**  
  ✅ Block-level performance  
  ✅ Remote access  
  ⚠️ Needs manual configuration and secure tunneling

### 3.2 📊 Repository Comparison Table

| Repository Type         | Fast Clone | Immutability | Remote-Ready | TLS |
|-------------------------|------------|--------------|--------------|-----|
| R3FS (Windows)          | ✅         | ❌           | ⚠️ (VPN req.) | ❌  |
| XFS (Linux)             | ✅         | ✅ (hardened) | ❌           | ❌  |
| S3 (MinIO/AWS/Wasabi)   | ❌         | ✅           | ✅           | ✅  |
| NAS (SMB/NFS)           | ❌         | ❌           | ✅           | ❌  |

🛡️   **Immutability Options**
- Linux Hardened Repo: ✅ (XFS + reflink)    
- S3 with Object Lock: ✅ (MinIO, AWS, Wasabi)
- Windows ReFS: ❌
- SMB/NFS: ❌

{{% admonition type="tip" title="Recommendations" %}}
- Use **Hardened Linux Repo** for on-prem immutable storage  
- Use **S3 with Object Lock** for offsite long-term protection  
- Avoid NAS or standard Windows ReFS for critical or immutable backup chains
{{% /admonition %}}

---

## 🚀 Backup Copy Jobs (Offsite Protection)

A **Backup Copy Job** is essential for offsite disaster recovery. It allows you to **replicate your primary backups** to a secondary location — protecting against ransomware, hardware failure, or site outages.

{{% admonition type="info" title="Use Case" %}}
You're backing up VMs to a local repository (e.g., Windows ReFS or NAS) and want to replicate them to an **offsite, immutable Hardened Linux Repository**.
{{% /admonition %}}

### 4.1 🛡 Why Use Hardened Linux

- ✅ **Immutable** storage with XFS + reflink
- 👤 Non-root user, no interactive shell
- 🔐 Resilient against ransomware — even if Veeam is compromised

### 4.2 🔧 Configuration Steps

{{% details summary="▶ Step 1: Prepare the Hardened Linux Repository" %}}

```bash
# Format the disk
sudo mkfs.xfs -m reflink=1 /dev/sdb
sudo mount /dev/sdb /mnt/veeam

# Create veeamrepo user
sudo useradd -m -s /sbin/nologin veeamrepo
sudo chown veeamrepo: /mnt/veeam
```

✅ Enable SSH key login
✅ Grant temporary sudo during setup
🔒 Disable password auth after configuration
{{% /details %}}

{{% details summary="▶ Step 2: Add Repository to Veeam" %}}

- Go to Backup Infrastructure > Backup Repositories > Add Repository
- Choose Linux Hardened Repository
- Enter:
  - IP address
  - SSH credentials
  - Public key path
  - Set mount path (e.g. /mnt/veeam)
  - Enable Immutability (e.g., 30 days)
{{% /details %}}

{{% details summary="▶ Step 3: Create the Backup Copy Job" %}}

- Navigate to Home > Backup Copy > Backup Copy Job (VMs)
- Select the primary job as source
- Choose Hardened Linux Repository as target
- Set:
  - Immediate copy mode: ✅ (optional)
  - Copy every 12 h, retain 7 restore points
{{% /details %}}

4.3 💡 Best Practices
{{% admonition type="tip" title="Best Practices" %}}

🧪 Enable Health Checks to validate copy integrity
🔄 Use WAN Accelerators if bandwidth is limited
⏳ Keep immutable retention longer than source (e.g., 30 days vs 14 days)
📬 Enable email alerts on failure
{{% /admonition %}}

4.4 📌 Summary
{{% admonition type="info" title="Summary" %}}

🔐 Backup Copy Jobs + Hardened Repo = Secure Offsite Copies
🧱 Protects against ransomware & data loss
☑️ Meets the 3-2-1-1-0 backup rule
🔁 Runs automatically, no scripting required
{{% /admonition %}}

---

## 📆 Scale-Out Backup Repository (SOBR)

A **SOBR** (Scale-Out Backup Repository) combines multiple backup repositories into a single logical unit. This allows flexible storage tiering, improved performance, and simplified management.

{{% admonition type="info" title="Use Case" %}}
Daily backups go to a local NAS (iSCSI, ReFS 64 KB), while older backups are moved to a remote MinIO S3 bucket with Object Lock enabled.
{{% /admonition %}}

### 5.1 🧱 Tiered Storage Strategy

**Performance Tier**
- 📍 Local NAS with iSCSI mount
- 📁 ReFS formatted with 64 KB block size
- ⚡ High-speed storage for daily restore operations

**Capacity Tier**
- 🌐 Remote MinIO S3 bucket
- 🔐 Object Lock + versioning enabled
- 🗄 Ideal for long-term archival and compliance with immutability

### 5.2 🔧 SOBR Configuration Steps

{{% details summary="▶ Step 1: Create Individual Repositories" %}}

- Format the local NAS volume with **ReFS 64 KB**
- Add it to Veeam as a **Direct Attached Repository**
- Deploy MinIO:
  - Enable **TLS**, **versioning**, and **Object Lock**
  - Create the S3 bucket for backups
- Add the bucket to Veeam as an **S3-Compatible Object Repository**
- Enable **immutability** (e.g., 30 days Governance mode)

{{% /details %}}

{{% details summary="▶ Step 2: Create and Configure the SOBR" %}}

- Navigate to `Backup Infrastructure > Scale-Out Repositories`
- Click **Add SOBR**
- Set a name (e.g., `SOBR-Primary-Remote`)
- Add:
  - Local ReFS repo as the **Performance Tier**
  - MinIO S3 repo as the **Capacity Tier**
- Enable the following options:
  - ✅ Move backups older than 14 days to capacity
  - ✅ Copy backups as soon as they are created (optional)
  - ✅ Support immutability on capacity tier

{{% /details %}}

### 5.3 📊 SOBR Benefits

{{% admonition type="tip" title="Why Use SOBR?" %}}

- 🚀 Automatically tier backups based on age or policy  
- 🌐 Combines high-performance local storage with remote S3 storage  
- 🔐 Enables compliance with immutability and offsite protection  
- ⚙️ Simplifies management without scripting

{{% /admonition %}}

---

## 🧠 Setting Up Efficient Backup Proxies

Backup Proxies in Veeam act as data movers between the source (e.g., hypervisors) and the backup repositories. A well-configured proxy infrastructure is essential for performance, scalability, and job reliability.

{{% admonition type="info" title="What Is a Proxy?" %}}
A backup proxy retrieves data from the source (e.g., VMware, Hyper-V), processes it (compression, deduplication, encryption), and sends it to the backup repository. Proxies reduce load on the main backup server.
{{% /admonition %}}

### 6.1 🧩 Default vs Additional Proxies

- By default, the Veeam Backup Server acts as a proxy
- In larger or distributed environments, **additional proxies** are highly recommended
- Multiple proxies allow **parallel job processing** and workload isolation

### 6.2 ⚙️ Add and Configure Proxies

{{% details summary="▶ Step 1: Add a New Proxy" %}}

- Open Veeam Backup & Replication Console
- Go to `Backup Infrastructure > Backup Proxies`
- Click **Add Proxy**
- Choose:
  - Platform: VMware vSphere or Microsoft Hyper-V
  - Server: Select existing Windows Server or deploy automatically

{{% /details %}}

{{% details summary="▶ Step 2: Configure Transport Mode" %}}

- Modes:
  - 🔄 **Automatic** – Veeam chooses the best mode
  - 📦 **Direct SAN Access** – best for fibre/iSCSI environments
  - 🔧 **Virtual Appliance (Hot-Add)** – efficient for virtual proxies
  - 🌐 **Network (NBD/NFS)** – fallback, least efficient

- Set maximum concurrent tasks based on CPU/RAM (e.g., 4–8)

{{% /details %}}

### 6.3 🔍 Monitoring and Scaling

{{% admonition type="tip" title="Proxy Scaling Tips" %}}

- Place proxies **close to production storage** for better performance  
- Use **dedicated proxies** instead of sharing with repositories  
- For distributed sites, deploy proxies **locally**  
- Monitor proxy usage in job stats and scale accordingly  
- Use **DRS affinity rules** in clusters to optimize job placement

{{% /admonition %}}

{{% admonition type="info" title="Recommended Platforms" %}}
- ✅ Windows Server 2019+ is stable and performant  
- 🧪 Linux proxies are available in experimental mode (CLI only)
{{% /admonition %}}

---

## 🌐 Optimizing WAN Transfers with Accelerators

Veeam WAN Accelerators optimize backup data transfer across slow or unreliable WAN links. They are especially useful for **Backup Copy Jobs** to remote sites or disaster recovery (DR) locations.

{{% admonition type="info" title="When to Use WAN Acceleration" %}}
Use WAN Accelerators when transferring backup data between sites with limited bandwidth, high latency, or inconsistent connectivity.
{{% /admonition %}}

### 7.1 🚀 What WAN Accelerators Do

- 🔁 **Global deduplication**: Avoids re-sending already transferred blocks  
- 🗂 **Global cache**: Stores repeated data patterns  
- 📦 **Compression & optimization**: Minimizes bandwidth usage  
- 📉 **Traffic reduction**: Up to 50x savings on repeated transfers

### 7.2 ⚙️ How to Configure WAN Acceleration

{{% details summary="▶ Step 1: Prepare the Accelerator Server" %}}

- Use a dedicated Windows Server (physical or VM)
- Recommended resources:
  - 2+ vCPU
  - 4+ GB RAM
  - Fast disk (SSD/NVMe) for cache (e.g., 40–100 GB)

{{% /details %}}

{{% details summary="▶ Step 2: Add WAN Accelerators in Veeam" %}}

- Go to `Backup Infrastructure > WAN Accelerators`
- Click **Add WAN Accelerator**
- Select your Windows server
- Define:
  - **Cache path** (dedicated volume preferred)
  - **Cache size** (recommend 1–3% of protected data)
  - Max concurrent tasks

Repeat this step for **both** source and target sites.

{{% /details %}}

{{% details summary="▶ Step 3: Use in Backup Copy Job" %}}

- Edit or create a Backup Copy Job
- In the **Data Transfer** step:
  - Enable **Use WAN Accelerators**
  - Select:
    - Source WAN Accelerator (e.g., local site)
    - Target WAN Accelerator (e.g., remote site)
- Complete job configuration

{{% /details %}}

### 7.3 💡 Best Practices

{{% admonition type="tip" title="Tips for Efficient WAN Transfers" %}}

- 🧠 Place WAN Accelerators **close to the repository**  
- 💽 Use a **separate disk** for cache — not shared with OS or backups  
- 🛡 Enable **TLS or VPN** if WAN is untrusted  
- 📊 Monitor **cache health and usage** via Veeam UI  
- 🧪 Test throughput before and after to confirm benefit

{{% /admonition %}}

{{% admonition type="info" title="Edition Requirement" %}}
WAN acceleration is available **only in Veeam Enterprise or Enterprise Plus** editions.
{{% /admonition %}}

---

## 📝 Final Recommendations

- ✅ Plan for 3-2-1-1-0 strategy.
- ✅ Always use immutability.
- ✅ Test restores monthly.
- ✅ Secure credentials and infrastructure.
- ✅ Enable alerts, health checks, and patch regularly.

A well-designed Veeam deployment delivers security, scalability, and disaster resilience — even for small environments.

