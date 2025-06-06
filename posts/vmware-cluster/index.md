# Creating and Configuring a VMware Cluster


## 1. Creating and Configuring a VMware Cluster

```goat
         ┌─────────-─────────┐         
         │   VMware Cluster  │         
         │ [DRS + HA + vSAN] │         
         └─────────┬─────────┘         
     ┌─────────────┼─────────────┐     
┌────▼───┐         ▼         ┌───▼────┐
│ ESXi 1 │       [...]       │ ESXi N │
└────────┘                   └────────┘
```

Creating a cluster in VMware vCenter is the foundational step for enabling features like High Availability (HA), Distributed Resource Scheduler (DRS), and vSAN. This section will guide you through the basic steps to create a production-ready cluster and connect ESXi hosts.

### 📌 Prerequisites

- A functioning **vCenter Server** instance.
- At least **2 ESXi hosts** (3+ recommended for full vSAN functionality).
- Proper IP connectivity between hosts and vCenter.
- DNS resolution working across the environment.

---

### 🛠 Step-by-Step: Cluster Creation

1. **Log into vCenter** using the vSphere Web Client.
2. In the **left navigation pane**, right-click your Datacenter and choose: New Cluster

3. Provide a meaningful **name** for your cluster (e.g., `Production-Cluster`).

4. Enable the following features:
- ✅ **DRS** – Automatically balances workloads between hosts.
- ✅ **HA** – Ensures VM restart in case of host failure.
- ✅ **vSAN** – Enables distributed storage (if applicable).

5. Click **Next** and complete the wizard.


---

### ➕ Adding Hosts to the Cluster

After the cluster is created:

- Select the **cluster name**, then click **Add Hosts**.
- Provide the **hostname/IP** of each ESXi server.
- Enter **root credentials** and validate the connection.
- Review and confirm **SSL thumbprint**.
- Finish the wizard to add the host.

Repeat this step for each ESXi host in your environment.

> ⚠️ If your ESXi hosts already contain virtual machines and vSwitches, ensure you **migrate networking and storage** configurations appropriately before or after adding them to the cluster.

---

### ✅ Initial Cluster Configuration

After adding the hosts, you can further configure the cluster:

- Configure **host licensing**.
- Review **network redundancy** (dual NICs recommended).
- Set up **vMotion and VMkernel adapters**.
- (Optional) Prepare for **vSAN or shared storage**.
- Enable **EVC** (Enhanced vMotion Compatibility) if mixing CPU generations.

---

### 🎯 Best Practices

- Name the cluster based on its function: `Mgmt-Cluster`, `App-Cluster`, `Test-Cluster`, etc.
- Always validate **hardware compatibility** via the VMware Compatibility Guide.
- Maintain **identical ESXi versions** across all hosts.
- Ensure **time synchronization** (NTP) is properly set across all nodes.

---

## 2. Configuring vDS and Network Design

```goat
                       ┌──────────────────────┐                      
                       │       vDS_Main       │                      
                       │ (Distributed Switch) │                      
                       │       MTU: 9000      │                      
                       └──────────┬───────────┘                      
                                  │                                  
                                  ▼                                  
                       ┌──────────────────────┐                      
                       │       Uplinks        │                      
                       │──────────┐───────────│                      
                       │ Uplink 1 │ Uplink 2  │                      
                       └──────────────────────┘                      
                            │            │                           
                            ▼            ▼                           
             ┌─────────────────────────────────────────┐             
             │               Port Groups               │             
             │─────────────────────────────────────────│             
             │ Name     VLAN MTU  │ Name     VLAN MTU  │             
             │ vDS_MGMT 10   1500 │ vDS_vMot 20   9000 │             
             │ vDS_WAN  100  1500 │ vDS_vSAN 30   9000 │             
             │                    │ vDS_LAN  200  1500 │             
             └─────────────────────────────────────────┘             
                                  │                                  
             ┌────────────────────┼────────────────────┐             
             │                    │                    │             
             ▼                    ▼                    ▼             
┌──────────────────────┐                     ┌──────────────────────┐
│      ESXI 1          │        [...]        │      ESXI N          │
│ vmnic0 -> Uplink 1   │                     │ vmnic0 -> Uplink 1   │
│ vmnic1 -> Uplink 2   │                     │ vmnic1 -> Uplink 2   │
│──────────────────────│                     │──────────────────────│
│ Port Group: vDS_MGMT │                     │ Port Group: vDS_MGMT │
│ VLAN ID :   10       │                     │ VLAN ID: 10          │
│ VMkernel Ports       │                     │ vMkernel Ports       │
│ vmk0 : 192.168.10.11 │                     │ vmk1 : 192.168.10.1N │
│ MTU: 1500            │                     │ MTU: 1500            │
│ Services: Management │                     │ Services: Management │
│──────────────────────│                     │──────────────────────│
│ Port Group: vDS_vMot │                     │ Port Group: vDS_vMot │
│ VLAN ID: 20          │                     │ VLAN ID: 20          │
│ vMkernel Ports       │                     │ vMkernel Ports       │
│ vmk1 : 192.168.20.11 │                     │ vmk1 : 192.168.20.1N │
│ MTU: 9000            │                     │ MTU: 9000            │
│ Services: vMotion    │                     │ Services: vMotion    │
│──────────────────────│                     │──────────────────────│
│ Port Group: vDS_vSAN │                     │ Port Group: vDS_vSAN │
│ VLAN ID: 30          │                     │ VLAN ID: 30          │
│ vMkernel Ports       │                     │ vMkernel Ports       │
│ vmk1 : 192.168.30.11 │                     │ vmk1 : 192.168.30.1N │
│ MTU: 9000            │                     │ MTU: 9000            │
│ Services:            │                     │ Services:            │
│ vSAN, vSAN Witness   │                     │ vSAN, vSAN Witness   │
└──────────────────────┘                     └──────────────────────┘
```

A Distributed Switch (vDS) provides centralized management of networking across all ESXi hosts in your cluster. It enables consistent configuration of port groups, VMkernel adapters, and uplinks. This section walks you through the setup of a minimal but robust networking environment.

---

### 🎯 Recommended Network Design (Minimal Topology)

For a basic production cluster with two ESXi hosts and one vCenter, the following roles should be isolated logically (or via VLANs):

| Traffic Type     | Interface Purpose       | Example VLAN | Example IP Address     |
|------------------|-------------------------|--------------|------------------------|
| Management       | vCenter & ESXi control  | VLAN 10      | `192.168.10.x`         |
| vMotion          | Live VM migration       | VLAN 20      | `192.168.20.x`         |
| vSAN             | Storage traffic         | VLAN 30      | `192.168.30.x`         |
| VM Network (LAN) | Internal services/apps  | VLAN 100     | `192.168.100.x`        |
| VM Network (WAN) | External/Internet-bound | VLAN 200     | `192.168.200.x`        |

Each host should have at least **2 physical NICs**. Use **NIC teaming** or dedicate one NIC per role (preferred for performance and fault tolerance).

---
### 🛠 Step-by-Step: Creating a vDS

1. In the **vSphere Web Client**, right-click the **Datacenter** → `Distributed Switch` → `New Distributed Switch`.

2. Set:
   - **Name**: `vDS_Main`
   - **Version**: Match your ESXi version (e.g., `8.0.0`)
   - **Number of Uplinks**: 2 (adjust if more NICs)
   - Enable Network I/O Control

3. Click **Finish**.

---

### ➕ Add Hosts to the vDS

1. Select the newly created `vDS_Main`.
2. Right-click → `Add and Manage Hosts` → `Add Hosts`.
3. Select the ESXi hosts from your cluster.
4. Map each **uplink** to a physical NIC on the host:
   - Uplink1 → `vmnic0`
   - Uplink2 → `vmnic1`

---

### 🌐 Create Distributed Port Groups

Each logical network gets its own port group.

| Name        | VLAN ID | Description                  |
|-------------|---------|------------------------------|
| `vDS_MGMT`  | 10      | Management / vCenter traffic |
| `vDS_VSAN`  | 30      | vSAN storage replication     |
| `vDS_VMOT`  | 20      | vMotion traffic              |
| `vDS_LAN`   | 100     | Internal VM traffic          |
| `vDS_WAN`   | 200     | External traffic (if needed) |

Use descriptive names for each port group and match the correct VLAN ID based on your switch configuration.

---

### 🧠 Configure VMkernel Adapters

For each ESXi host:

1. Go to `Host` → `Configure` → `VMkernel Adapters` → `Add VMkernel Adapter`.

2. Create one adapter per role:

| Port Group  | IP Address        | Services Enabled              |
|-------------|-------------------|-------------------------------|
| vDS_MGMT    | `192.168.10.11`   | ✅ Management                 |
| vDS_VMOT    | `192.168.20.11`   | ✅ vMotion                    |
| vDS_VSAN    | `192.168.30.11`   | ✅ vSAN                       |

3. Use separate subnets and gateways where required. The **vSAN** and **vMotion** interfaces typically have **no default gateway**.

Repeat for other hosts with:
- `.12` for Host 2
- `.13` for Host 3 (if any)

---

### 💡 Best Practices

- Separate critical services (Management, vSAN, vMotion) into different **VLANs or physical NICs**.
- Use **LACP or Active/Standby NIC teaming** for redundancy.
- Enable **Network I/O Control** to prioritize traffic types.
- Always label your Port Groups clearly — consistent naming saves time during troubleshooting.
- For 3+ hosts, consider dedicating NICs solely for vSAN performance.

---

Next up, we’ll dive into setting up **vSAN**, including disk claiming, witness host deployment, and storage policy configuration.

## 3. vSAN Setup and Configuration

```goat
                                           VMware vSAN                                            
                                 (Virtual Storage Area Network)                                   
                               Stretched Cluster and Fault Domains                                
                                  ┌────────────────────────────┐                                  
                                  │     Witness Appliance      │                                  
                                  │ Witness holds metadata only│                                  
                       ┌──────────┼       (quorum vote)        ┼─────────┐                        
                       │          └────────────────────────────┘         │                        
              ┌────────▼─────────┐                              ┌────────▼─────────┐              
              │      Site A      │      RAID-1 Replication      │      Site B      │              
              │ (Fault Domain A) │      Between Sites A/B       │ (Fault Domain B) │              
              └────────┬─────────┘                              └────────┬─────────┘              
       ┌───────────────┼───────────────┐                 ┌───────────────┼───────────────┐        
       │               │               │                 │               │               │        
┌──────▼───────┐┌──────▼───────┐┌──────▼───────┐  ┌──────▼───────┐┌──────▼───────┐┌──────▼───────┐
│ ESXi Host 1A ││ ESXi Host 2A ││ ESXi Host 3A │  │ ESXi Host 1B ││ ESXi Host 2B ││ ESXi Host 3B │
│──────────────││──────────────││──────────────│  │──────────────││──────────────││──────────────│
│ Local Disks  ││ Local Disks  ││ Local Disks  │  │ Local Disks  ││ Local Disks  ││ Local Disks  │
│ - SSD / HDD  ││ - SSD / HDD  ││ - SSD / HDD  │  │ - SSD / HDD  ││ - SSD / HDD  ││ - SSD / HDD  │
└──────┬───────┘└──────┬───────┘└──────┬───────┘  └──────┬───────┘└──────┬───────┘└──────┬───────┘
       │               └────────────┐  │                 │  ┌────────────┘               │        
       └─────────────────────────┐  │  │                 │  │  ┌─────────────────────────┘        
                               ┌─▼──▼──▼─────────────────▼──▼──▼─┐                                
                               │           vSAN Datastore        │                                
                               │       for all VMs in cluster    │                                
                               └────────────▲────▲────▲──────────┘                                
                      ┌──────────────┐      │    │    │       ┌──────────────┐                    
                      │ VM on Host 1 ┼──────┘    │    └───────┼ VM on Host N │                    
                      └──────────────┘         [...]          └──────────────┘                    
```

VMware vSAN pools local disks from each ESXi host to create a resilient, shared datastore — enabling a hyperconverged infrastructure.

---

### 🔍 vSAN Architecture: ESA vs OSA

```goat
              VSAN OSA                       VSAN ESA              
  (Original Storage Architecture)   (Express Storage Architecture) 
                                                                   
          ┌─────────────┐                  ┌─────────────┐         
          │  ESXi Host  │                  │  ESXI Host  │         
          └──────┬──────┘                  └──────┬──────┘         
       ┌─────────┼──────────┐                     ▼                
       ▼         ▼          ▼              ┌─────────────┐         
┌────────────┐       ┌─────────────┐       │ All-Flash   │         
│ Disk Group1│ [...] │ Disc GroupN │       │ Storage Pool│         
└──────┬─────┘       └──────┬──────┘       └─────────────┘         
       ▼                    ▼                                      
┌────────────┐SSD/NVMe┌────────────┐                               
│ Cache Tier │<- 1x ->│ Cache Tier │                               
└──────┬─────┘        └─────┬──────┘                               
       ▼                    ▼                                      
    HDD/SDD              HDD/SDD                                   
┌────────────────┐┌────────────────┐                               
│ Capacity Disk1 ││ Capacity Disk1 │                               
│────────────────││────────────────│                               
│     [...]      ││      [...]     │                               
│────────────────││────────────────│                               
│ Capacity DiskN ││ Capacity DiskN │                               
└────────────────┘└────────────────┘                               
```

| Feature                      | OSA (Original Storage Arch.) | ESA (Express Storage Arch.)      |
|-----------------------------|------------------------------|----------------------------------|
| Introduced in               | vSAN 5.5                     | vSAN 8.0                         |
| Cache/Capacity Separation   | Required                     | Not required                     |
| Performance Optimization    | Manual                       | Built-in, adaptive               |
| Disk Requirements           | SSD + HDD                    | All NVMe/SSD only                |
| Licensing                   | Enterprise Plus              | Enterprise Plus (w/ ESA support) |

**Recommendation**: If you're using modern NVMe disks and vSAN 8+, choose **ESA** for simplicity and better performance.

---

### 📦 Hardware Requirements

- Minimum: **3 ESXi hosts**, or **2 hosts + 1 Witness**
- Each host must have:
  - At least **1 SSD/NVMe** for cache (OSA)
  - At least **1 HDD** or SSD for capacity
- All hosts must have **same ESXi version**
- **RAID controllers** in passthrough or RAID0 mode (JBOD)
- Consistent **network config and IP ranges**

---

### 🛠 Step-by-Step: Enable vSAN

1. Go to `Cluster` → `Configure` → `vSAN` → `Services`
2. Click **Configure**
3. Choose **vSAN Type**:
   - `vSAN HCI` – Most common
   - `vSAN Max` – Disaggregated storage pool
4. Cluster topology:
   - ✅ `Two node cluster` – for edge or small sites
   - ✅ `Single site cluster` – standard 3+ host deployment
5. Choose architecture:
   - ✅ **vSAN ESA** (if supported)
   - or ❌ OSA (legacy method)

---

### ⚙️ Feature Configuration

| Option                   | Recommendation                  |
|--------------------------|----------------------------------|
| Compression              | ✅ Enabled (ESA only)           |
| Deduplication            | ❌ Optional (OSA only)          |
| Encryption at Rest       | ❌ Optional                     |
| Encryption in Transit    | ✅ (default from vSAN 7 U1)     |
| Allow reduced redundancy | ✅ Useful for maintenance mode  |
| RDMA support             | ❌ Unless special HW used       |

---

### 🌐 Fault Domains & Stretched Cluster Configuration

vSAN allows for advanced availability setups using **Fault Domains** and **Stretched Clusters**. These features ensure data resilience even during site or rack-level failures.

#### 🧱 Fault Domains (FDs)

- Used to group ESXi hosts that might **fail together**, such as those in the same rack, power domain, or chassis.
- Ensures that vSAN places **replica components** across **different FDs**, not just different hosts.
- Configure FDs under `Cluster → Configure → vSAN → Fault Domains`.

| Use Case                   | Benefit                                |
|----------------------------|----------------------------------------|
| Hosts in same rack         | Protect against rack-level failure     |
| Edge deployments           | Use 2-node with Witness as 3rd FD      |
| Larger clusters (6+ hosts) | Improve replica distribution           |

> 💡 At least **3 Fault Domains** are required for RAID-1 with full availability.

---

#### 🌍 Stretched Cluster

- A special vSAN topology designed for **geographically separated sites**.
- Consists of:
  - **2 Active Fault Domains** (Site A and Site B)
  - **1 Witness Appliance** (remote, vote-only, no data)

| Component         | Role                         |
|-------------------|------------------------------|
| Site A            | Half of data & compute       |
| Site B            | Other half of data & compute |
| Witness           | Maintains quorum             |

vSAN automatically replicates VM data **between sites (RAID-1)** and uses the Witness to avoid split-brain scenarios.

> 📝 Witness must be **outside the two sites**, reachable by both.

---

#### 🧪 Stretched Cluster Example: RAID-1 or RAID-5

You can choose storage policies per VM:

| Policy Type          | Description                              |
|----------------------|------------------------------------------|
| RAID-1 (Mirroring)   | Best for performance, uses more space    |
| RAID-5 (Erasure Code)| Space-efficient, needs 4+ data hosts     |

- RAID-1 across sites ensures **1:1 mirroring**
- RAID-5 is **intra-site only**, used within each fault domain

---

#### ✅ Best Practices

- Ensure low latency (≤ 5ms RTT) between active sites.
- Use **dedicated VMkernel interfaces** for vSAN on separate VLANs.
- Witness traffic should be isolated if possible.
- Monitor `vSAN → Health → Stretched Cluster` regularly.
- Use **Stretched Cluster only when geographic redundancy is required** — otherwise, configure simple Fault Domains.

---

### 💾 Disk Claiming

vCenter will automatically detect eligible disks.

| Tier        | Device Type     | Example Size | Notes                        |
|-------------|------------------|--------------|------------------------------|
| Cache Tier  | SSD / NVMe       | 250 GB+      | Only in OSA mode             |
| Capacity    | HDD / SSD        | 500 GB+      | All disks in ESA             |

- Assign disks to appropriate tiers.
- Confirm all hosts contribute capacity to the cluster.

---

### 🧪 Example: 2-node + Witness Topology

```goat
                 +-------------------+
                 | Witness Appliance |
                 | 192.168.10.100    |
                 +-------------------+
                           |
                           |
+---------------+          |          +---------------+
|    ESXI 1     |<---------+--------->|     ESXI 2    |
| 192.168.10.11 |                     | 192.168.10.12 |
| 192.168.30.11 |      vSAN Links     | 192.168.30.12 |
+---------------+                     +---------------+
```

| Node       | Management IP   | vSAN IP        | Role              |
|------------|-----------------|----------------|-------------------|
| ESXI 1     | 192.168.10.11   | 192.168.30.11  | Data + Compute    |
| ESXI 2     | 192.168.10.12   | 192.168.30.12  | Data + Compute    |
| Witness VM | 192.168.10.100  | 192.168.30.100 | Voting only node  |

> 📝 Witness Appliance is a **lightweight virtual machine**. Deploy it using **OVF template** and place it **outside the vSAN cluster**.

---

### 🧰 Create vSAN Storage Policy

1. Go to `Policies and Profiles` → `VM Storage Policies`
2. Create a new policy: `vSAN-Standard`
3. Configure:

```text
Name: vSAN-Standard
vSAN rules:
- Availability: "2 failures – RAID-1 (Mirroring)"
- Object space reservation: Thin provisioning
- Flash read cache: 0%
- Force provisioning: Yes
```

You can also create "No Redundancy" policies for single-host test environments.

🏁 Result: Shared Resilient Datastore
After completing the wizard:

A new vSAN Datastore appears under Datastores
All eligible VMs can now be placed on vSAN
Storage policies can be assigned during VM deployment or migration

💡 Best Practices
Use separate VMkernel interfaces for vSAN traffic (VLAN + no gateway).
Ensure jumbo frames are enabled (MTU 9000) if network supports it.
Avoid mixing disk types or vendors within a cluster.
Monitor health under vSAN → Health
Enable troubleshooting logging if issues arise

## 4. Configuring High Availability (HA)

```goat
Initial State:
+--------+               +--------+
| ESXI 1 |               | ESXI 2 |
|  VM1   |               |  idle  |
+--------+               +--------+

After Host A fails:
+--------+               +--------+
| ESXI 1 |      -->      | ESXI 2 |
| down   |      VM1      |  VM1   |
+--------+               +--------+
```

vSphere High Availability (HA) protects your virtual machines by restarting them on another ESXi host in the cluster if a failure occurs. HA is a critical feature in production environments to minimize downtime.

---

### 🎯 What Does HA Protect Against?

| Failure Type          | HA Protection | Explanation                                     |
|-----------------------|---------------|-------------------------------------------------|
| ESXi Host Failure     | ✅ Yes        | VM restarts on surviving host                   |
| VM Crash              | ❌ No         | Requires VMware FT or 3rd-party monitoring      |
| Storage Failure       | ⚠️ Limited     | Depends on vSAN resilience                      |
| Network Partition     | ✅ Yes        | Uses Datastore or vSAN heartbeat                |

---
### 🛠 Step-by-Step: Enabling HA

1. Go to your **cluster** → `Configure` → `vSphere Availability`.
2. Click **Edit**.
3. Enable:
   - ✅ **Turn on vSphere HA**
   - (Optional) Enable **Admission Control**
4. Under "Failure conditions and responses":
   - Host Failure Response: `Restart VMs`
   - Isolation Response: `Power off and restart VMs`
5. Click **OK** to apply.
---

### 🧠 Admission Control Explained

Admission Control ensures that the cluster reserves enough resources to restart VMs if a host fails.

| Policy Type                         | Description                                     |
|-------------------------------------|-------------------------------------------------|
| Slot Policy                         | Based on largest CPU/memory reservation         |
| Cluster Resource Percentage         | Recomm. – reserve e.g., 33% CPU & memory        |
| Dedicated Failover Hosts            | Explicitly reserve 1 or more hosts              |

**Recommendation**: Use **Percentage-Based** policy for flexibility and better resource utilization.

---

### ⚙️ Advanced HA Options

- **Heartbeat Datastores**:
  - Used to confirm host is alive during network isolation.
  - Enabled automatically with vSAN.
- **VM Monitoring**:
  - Restarts VMs if VMware Tools stops responding.
  - Good for application crash recovery.

Enable this under:
Cluster → Configure → vSphere Availability → VM Monitoring


- **Proactive HA** (requires vendor plugin):
  - Migrates VMs off hosts reporting hardware degradation (e.g., failed DIMMs, overheating)

---

### 🛡 Recommended Settings

| Setting                       | Value                           |
|-------------------------------|---------------------------------|
| Host Monitoring               | ✅ Enabled                      |
| VM Monitoring                 | ✅ Enabled (medium sensitivity) |
| Isolation Response            | Power off and restart VMs       |
| Datastore Heartbeating        | ✅ Use multiple datastores      |
| Admission Control Policy      | 33% reserved resources          |

---

### 🔍 Example Scenario

You have a 3-node vSAN cluster:
- 1 host fails → HA restarts VMs from that host on remaining 2
- If **Admission Control** is enabled, only a safe number of VMs are allowed to power on initially to guarantee restart in case of failure

---

### 💡 Best Practices

- Always test HA by performing a **host shutdown** (not reboot).
- Monitor HA events in `Events` or `Tasks` tabs.
- Don’t disable **Host Monitoring** unless absolutely required (e.g., during firmware upgrades).
- Combine with **DRS** to ensure intelligent VM placement post-restart.

---

In the next section, we will explore **Distributed Resource Scheduler (DRS)** — VMware’s intelligent load balancer for VM workloads.

## 5. Configuring DRS (Distributed Resource Scheduler)

```goat
Before DRS:
+----------+       +----------+       +----------+
| ESXI 1   |       | ESXI 2   |       | ESXI 3   |
| LOAD 90% |       | LOAD 10% |       | LOAD 45% |
| VM1      |       |          |       | VM3      |
| VM2      |       |          |       |          |
+----------+       +----------+       +----------+

After DRS Migration:
+----------+       +----------+       +----------+
| ESXI 1   |  -->  | ESXI 2   |       | ESXI 3   |
| LOAD 60% |  VM2  | LOAD 40% |       | LOAD 45% |
| VM1      |       | VM2      |       | VM3      |
+----------+       +----------+       +----------+
```

VMware Distributed Resource Scheduler (DRS) is responsible for automatically balancing compute workloads (CPU and memory) across hosts in a cluster. It helps optimize performance and resource utilization with minimal manual intervention.

---

### 🎯 What DRS Does

- **Monitors resource usage** (CPU, RAM) across ESXi hosts
- **Live migrates** VMs (via vMotion) to balance loads
- **Prioritizes critical workloads** and improves availability

DRS works **only if vMotion** is properly configured between hosts and there is **shared storage** (like vSAN or NFS).

---

### 🛠 Step-by-Step: Enabling DRS

1. Go to your **cluster** → `Configure` → `vSphere DRS`.
2. Click **Edit**.
3. Enable:
   - ✅ **Turn on vSphere DRS**
4. Set Automation Level:
   - 🔘 **Fully Automated** (recommended)
   - ⭕ Partially Automated (for initial placement only)
   - ⭕ Manual (you approve every migration)
5. Optionally set Migration Threshold:
   - From Conservative (fewer moves) → Aggressive (more frequent balancing)
6. Click **OK**.

---

### 🧪 Example Scenario

🛠 Example Scenario (Why Partially Automated is Safer)
You have 3 hosts:

Host01: Heavy load (90% CPU)
Host02 / Host03: Idle or lightly loaded
In Fully Automated mode:
→ DRS might vMotion critical database VMs without your knowledge.

In Partially Automated mode:
→ DRS still recommends rebalancing, but you decide when and what to move.

This is essential in environments where:

Apps are sensitive to vMotion timing,
Licensing or performance isolation matters,
You need predictability during peak hours.

---

### ⚙️ Advanced DRS Options

| Feature                        | Description                                      |
|--------------------------------|--------------------------------------------------|
| VM/Host Affinity Rules         | Keep VMs together or apart (e.g., DB + App tiers) |
| Host Groups                    | Define sets of hosts for special placement        |
| DRS Score (vSphere 7+)         | Shows VM happiness based on resource availability |
| DRS for Maintenance Mode       | Automatically migrates VMs off host              |
| Predictive DRS (via vROps)     | Uses analytics to anticipate load                |

---

### ⚖️ Automation Level Explained

| Mode              | Behavior                          | Use Case                                  |
|-------------------|-----------------------------------|-------------------------------------------|
| Manual            | Shows recommendations only        | Training/testing environments             |
| Partially Auto    | ✅ Automatic initial placement    | 🔒 Conservative or regulated operations   |
|                   | 🔍 Manual approval for migrations | 🛠 Controlled production environments     |
| Fully Automated   | Initial + ongoing migrations      | Lab/testing or low-risk workloads         |

> 🛡️ Recommended Mode: Partially Automated is recommended in most production environments. It provides automation without losing control, ensuring performance without risking unexpected migrations.

---

### 🛡 Recommended Settings

| Setting                  | Value                         |
|--------------------------|-------------------------------|
| DRS Mode                 | ✅ Partially Auto             |
| Migration Threshold      | Level 3 or 4 (Balanced)       |
| Affinity Rules           | Define as needed              |
| Host Groups              | Use for licensing or zones    |
| Predictive DRS           | Optional (requires vROps)     |

---

### 💡 Best Practices
- Combine DRS with HA for maximum resiliency.
- Monitor DRS recommendations under Cluster → Monitor → DRS.
- Regularly review affinity/anti-affinity rules.
- For multi-tier apps (e.g., Web + DB), use anti-affinity to separate roles.
- Enable EVC for mixed CPU environments to ensure vMotion compatibility.

---

With HA and DRS properly configured, your cluster can now **balance workloads** and **survive hardware failures** — fully automated, fully redundant.

---

### ✅ Summary of Cluster Setup

| Feature | Purpose                             | Configured in Section |
|--------|--------------------------------------|------------------------|
| Cluster | Aggregates hosts & enables features | 1                      |
| vDS     | Centralized network configuration   | 2                      |
| vSAN    | Shared hyperconverged storage       | 3                      |
| HA      | VM restart on host failure          | 4                      |
| DRS     | Load balancing via vMotion          | 5                      |

---

> 🧩 This completes your production-ready VMware cluster setup. If you'd like to export a full configuration checklist or Markdown version for documentation or GitHub repo, let me know!




