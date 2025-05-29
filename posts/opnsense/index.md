# OPNsense Setup: Basic Guide


[OPNsense](https://opnsense.org/) is a powerful open-source firewall and routing platform based on FreeBSD.

---

## 1.1 Installing as VM

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

### 1.2 First Login to OPNsense

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

### 1.3 Configure WEB access from WAN
 
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

### 1.4 System Updates and Backup Configuration

> 👉 System > Firmware > Status

Click **Check for updates** → then **Apply updates**.

> 👉 System > Configuration > Backups

Download a local `.xml` backup of your configuration.  
You can also enable automatic backups to cloud services like Google Drive, Nextcloud, etc.

---

### Useful Resources

- 📚 [Official OPNsense Documentation](https://docs.opnsense.org/)
- 🎥 [YouTube: OPNsense Tutorials](https://www.youtube.com/results?search_query=opnsense)

---

## 2.1 WireGuard Site-to-Site VPN Setup

WireGuard is a fast and modern VPN solution known for its simplicity and performance.

✅ What You’ll Need

  - Two OPNsense firewalls with internet access (Site A and Site B)
  - Static or dynamic public IPs (DDNS works)
    - Site A -> WAN IP 203.0.113.1 and LAN IP 192.168.1.1/24
    - Site B -> WAN IP 203.0.113.2 and LAN IP 192.168.2.1/24
  - WireGuard plugin installed (os-wireguard)

---

### 2.2 Create WireGuard Instance

> 👉 VPN > WireGuard > Instances

  - Click **+ Add** to create a new instance.
    - [x] Enabled
    - Name: **site-a** on Site A / **site-b** on Site B
    - Click ⚙️  `Generate new keypair`
    - Listen Port **51820**
    - Tunnel Address 
      - Site A: **10.2.2.1/24**
      - Site B: **10.2.2.2/24**
    - Peers: it will be needed after peer created

💾 Press Save and Apply

⚠️  After saving, copy the Public Key. You'll need it on the remote site.

### 2.3 Configure Peer

> 👉 VPN > WireGuard > Peers

  - Click **+ Add** to create a new peer.
    - [x] Enabled
    - Name: **site-b** on Site A / **site-a** on Site B
    - `Public Key` of the remote site
    - Allowed IPs: remote local subnet and Instance IP
      - On Site A: **10.2.2.2/32** **192.168.2.0/24**
      - On Site B: **10.2.2.1/32** **192.168.1.0/24**
    - Endpoint Address: remote WAN IP (or DDNS)
      - On Site A: **203.0.113.2**
      - On Site B: **203.0.113.1**
    - Endpoint Port: **51820**

💾 Press Save and Apply

⚠️  Now go back to Instances, edit your instance, and add this peer to Peers.

💾 Press Save and Apply

  - [x] Enable WireGuard and 💾 Press Apply

### 2.4 Create Interfaces

> 👉 Interfaces > Assignments

  - Assign the new WireGuard interface (wg0)
  - Enable it:
    - Set name: WG_SITEA / WG_SITEB

💾 Press Save and Apply

### 2.5 Set Up Firewall Rules

> 👉 Firewall > Settings > Normalization

  - Interface: **WireGuard (Group)**
  - Direction: **Any**
  - Protocol: **any**
  - Source: **any**
  - Destination: **any**
  - Destination port: **any**
  - Description: **Wireguard MSS Clamping**
  - Max mss: **1380** or lower, subtract at least 40 bytes from the Wireguard MTU

> 👉 Firewall > Rules > WAN

  - Action: **Pass**
  - Interface: **WAN**
  - Direction: **In**
  - TCP/IP Version: **IPv4**
  - Protocol: **UDP**
  - Source: Remote WAN IP
    - On Site A: **203.0.113.1**
    - On Site B: **203.0.113.2**
  - Destination: **WAN address**
  - Destination port: **51820**
  - Description:
    - On Site A: **Allow Wireguard from Site B to Site A**
    - On Site B: **Allow Wireguard from Site A to Site B**

> 👉 Firewall > Rules > LAN

  - Action: **Pass**
  - Interface: **LAN**
  - Direction: **In**
  - TCP/IP Version: **IPv4**
  - Protocol: **Any**
  - Source: Remote LAN IP
    - On Site A: **192.168.2.0/24**
    - On Site B: **192.168.1.0/24**
  - Destination: **LAN net**
  - Destination port: **51820**
  - Description: 
    - On Site A: **Allow LAN Site B to LAN Site A**
    - On Site B: **Allow LAN Site A to LAN Site B**


> 👉 Firewall > Rules > Wireguard (Group)

  - Action: **Pass**
  - Interface: **Wireguard (Group)**
  - Direction: **In**
  - TCP/IP Version: **IPv4**
  - Protocol: **Any**
  - Source: Remote LAN IP
    - On Site A: **192.168.2.0/24**
    - On Site B: **192.168.1.0/24**
  - Destination: **LAN net**
  - Destination port: **51820**
  - Description:
    - On Site A: **Allow LAN Site B to LAN Site A**
    - On Site B: **Allow LAN Site A to LAN Site B**

> 👉 VPN > WireGuard > Status

  ✅ Check for handshake and endpoint status.

> 👉 Interfaces > Diagnostics > Ping

  - Try pinging remote LAN IP (e.g., from 192.168.1.1 → 192.168.2.1)

---

### Useful Resources

- 📚 [Official OPNsense Documentation](https://docs.opnsense.org/manual/how-tos/wireguard-s2s.html)

---

## 3.1 IPsec VPN for iOS and macOS

This guide walks you through the setup of an IPsec VPN server on **OPNsense** that works with **iOS** and **macOS** devices.  
Tested with:  
- **OPNsense** 23.1.7_3 – 24.7.5_3  
- **iOS** 16+  
- **macOS** Monterey+

---

### 👥 3.1 (Optional) Create a VPN Group

**System → Access → Groups → Add**

- **Group name:** VPN  
- **Description:** VPN users

---

### 👤 3.2 Create a VPN User

**System → Access → Users → Add**

- **Username:** vpnuser  
- **Password:** [secure password]  
- **Member Of:** VPN (optional, if you created the group)

---

### 🌐 3.3 Configure IPsec Mobile Clients

**VPN → IPsec → Mobile Clients**

- ✅ Enable IPsec Mobile Client Support  
- **Backend for authentication:** Local Database  
- **Enforce local group:** VPN (optional)  

 IP Address Pool
- ✅ Provide a virtual IPv4 address  
- **Address Pool:** 192.168.2.0/24  

 DNS Servers
- ✅ Provide DNS server list  
  - Server 1: 192.168.1.1 (local router IP)  
  - Server 2: 8.8.8.8  
  - Server 3: 8.8.4.4  

- **Phase 2 PFS Group:** Off  
- Save and Apply changes

---

### 🔑 3.4 Create Phase 1

Click **"Create Phase1"** at the top blue bar.

- **Connection method:** default  
- **Key Exchange version:** auto  
- **Interface:** WAN  
- **Description:** Mobile VPN  
- **Authentication:** Mutual PSK + Xauth  
- **My identifier:** My IP address  
- **Pre-Shared Key:** [store securely]

Algorithms
- AES 256  
- Hash: SHA256, SHA384, SHA512  
- DH group: 2, 5, 14–18  
- ✅ Install policy  
- NAT Traversal: Enable  
- Lifetime: 28800  
- Save and Apply changes

---

### 🔄 3.5 Create Phase 2

**VPN → IPsec → Tunnel Settings**  
Click ➕ in **Commands** column of Phase 1 row.

- **Mode:** Tunnel IPv4  
- **Type:** Network  
- **Address:** 0.0.0.0/0  
- **Protocol:** ESP  
- **Encryption:** AES128, AES192, AES256  
- **Hash:** SHA256, SHA384, SHA512  
- **PFS:** Off  
- Lifetime: 3600  
- Save and Apply

---

### ✅ 3.6 Enable IPsec

**VPN → IPsec → Tunnel Settings**

- ✅ Enable IPsec (bottom of page)  
- Apply changes

---

### 🔥 3.7 Configure Firewall: WAN Rules

**Firewall → Rules → WAN → Add 3 rules:**

1. **Protocol:** ESP → Destination: WAN → Description: IPsec ESP  
2. **Protocol:** TCP/UDP → Port: ISAKMP (500) → Description: IPsec ISAKMP  
3. **Protocol:** UDP → Port: NAT-T (4500) → Description: IPsec NAT-T  

✅ Apply changes

---

### 🔒 3.8 Configure Firewall: IPsec Rules

**Firewall → Rules → IPsec**

- Protocol: Any  
- Source: Any  
- Destination: Any  

✅ Apply changes

---

### 🌍 3.9 Outbound NAT

**Firewall → NAT → Outbound**

- Mode: **Automatic outbound NAT**  
- Confirm `192.168.2.0/24` is listed

---

### 📡 3.10 Allow DNS from VPN

**Services → Unbound DNS → Access Lists → Add**

- **Access List name:** 192.168.2.0/24  
- **Action:** Allow  
- **Network:** 192.168.2.0/24  
- **Description:** VPN DNS access  
- Save and Apply

---

### 🔄 3.11 (Optional) Allow Multi-Device Connections

In **Phase 1**, set:

- **Unique ID:** `no`  
This allows one user to connect from multiple devices.

---

### 📲 3.12 iPhone VPN Configuration

**Settings → VPN → Add VPN Configuration…**

- **Type:** IPsec  
- **Server:** [OPNsense public IP]  
- **Account:** vpnuser  
- **Password:** [user password]  
- **Secret:** [Pre-Shared Key]  
- Leave "Group Name" empty

---

### 💻 3.13 macOS VPN Configuration

**System Preferences → Network → + → Interface: VPN**

- **VPN Type:** Cisco IPSec  
- **Server Address:** [OPNsense public IP]  
- **Account Name:** vpnuser  
- **Password:** [user password]  
- **Shared Secret:** [Pre-Shared Key]  

✅ Save and Connect

---

### 💡 Troubleshooting

- If DNS doesn’t work — try restarting the router  
- Double-check firewall and mobile client settings  
- Use strong, unique Pre-Shared Keys and passwords

---

*📝 Published on [blog.topli.ch](https://blog.topli.ch/network/opnsense/)*


