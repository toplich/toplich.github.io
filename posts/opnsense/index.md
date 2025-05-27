# OPNsense Setup: Basic Guide


[OPNsense](https://opnsense.org/) is a powerful open-source firewall and routing platform based on FreeBSD.

---

## Installing as VM

Download the [ISO](https://opnsense.org/download/)

  - Architecture: **amd64**
  - Image type: **DVD ISO**

General VM Settings

   - OS Type → **FreeBSD (64-bit)**

|  Option         |  Minimum    |  My conf  |  Recommended  |  
|  -------------  | ----------- | --------- | ------------- |
|  CPU            |  1 cores    |  2 cores  |  multi core   |
|  Memory         |  512 MB     |  2 GB     |  ≥ 4 GB       |
|  Disk Size      |  8 GB       |  16 GB    |  120 GB       |

First-time login credentials:

  - **Username:** `installer`
  - **Password:** `opnsense`

After logging in, the system will automatically launch the text-based installer.

  1. Accept license and keyboard layout
  2. Choose Install (UFS) or (ZFS)
  3. Select target disk
  4. Set a new password for the root user
  5. Wait for installation to complete
  6. Remove the ISO and reboot

---

### 1. First Login to OPNsense

After reboot, you can log: 
 - From LAN network: <https://opnsense.localdomain> or <https://192.168.1.1>
 - From WAN we need disable Firewall (Shell): `pfctl -d`

Default credentials:
  - **Username:** `root`
  - **Password:** `opnsense`

> 👉 System > Wizard
 - General Setup

    1. Set hostname, domain (optional), and DNS servers.
    2. Select your time zone.
    3. Configure interfaces (WAN / LAN).
    4. Set a new root password.

---

### 2. Configure WEB access from WAN
 
> 👉 Firewall > Rules > WAN
 - add new Rule
   - Protocol: **TCP**
   - Destination: **WAN address**
   - Destination port range
     - From: **(other) 8080** 
     - To: **(other) 8080**

> 👉 System > Settings > Administration
  - Protocol: **HTTPS access**
  - TCP Port: **8080**

> 👉 System > Firmware > Plugins
 - Install **os-vmware**

---

### 3. System Updates and Backup Configuration

> 👉 System > Firmware > Status

Click **Check for updates** → then **Apply updates**.

> 👉 System > Configuration > Backups

Download a local `.xml` backup of your configuration.  
You can also enable automatic backups to cloud services like Google Drive, Nextcloud, etc.

---

## Useful Resources

- 📚 [Official OPNsense Documentation](https://docs.opnsense.org/)
- 🎥 [YouTube: OPNsense Tutorials](https://www.youtube.com/results?search_query=opnsense)

---

*📝 Published on [blog.topli.ch](https://blog.topli.ch/network/opnsense/)*


