# 🏠 Homelab Infrastructure Guide
## Secure Server & Network Setup — Complete Implementation Plan

> **Hardware Inventory**
> - Hypervisor Host (Proxmox/ESXi) with **≥ 2 NICs**
> - Virtual Sophos Firewall (VM)
> - HPE Aruba Instant On 1930 — 48-Port Managed Switch
> - D-Link 8-Port Gigabit Unmanaged Switch
> - 1× 2 TB SSD · 1× 4 TB HDD

---

## 1. Network Topology & Architecture

### 1.1 Physical Cabling Diagram

```
                        ┌─────────────┐
                        │  ISP Modem   │
                        │  (Bridge Mode)│
                        └──────┬───────┘
                               │ Ethernet (WAN)
                               ▼
                  ┌────────────────────────┐
                  │    HYPERVISOR HOST      │
                  │  ┌──────────────────┐  │
                  │  │  NIC 1 (WAN)     │◄─┘  (Dedicated to Sophos WAN)
                  │  └──────────────────┘  │
                  │  ┌──────────────────┐  │
                  │  │  NIC 2 (LAN)     │──┼───► Trunk to Aruba Switch
                  │  └──────────────────┘  │
                  │                        │
                  │  ┌──────────────────┐  │
                  │  │  Sophos XG VM    │  │     (Virtual Firewall)
                  │  │  vNIC0 → NIC 1   │  │
                  │  │  vNIC1 → NIC 2   │  │
                  │  └──────────────────┘  │
                  │                        │
                  │  ┌──────────────────┐  │
                  │  │  Other VMs       │  │     (Connected via vSwitch/bridge)
                  │  └──────────────────┘  │
                  └────────────────────────┘
                               │
                               │ 802.1Q Trunk (all VLANs tagged)
                               ▼
                  ┌────────────────────────┐
                  │  HPE Aruba 1930        │
                  │  48-Port Managed Switch│
                  │                        │
                  │  Port 1:  Trunk ← Host │
                  │  Port 2-24: Access     │     (Servers, Workstations, APs)
                  │  Port 25-47: Access    │
                  │  Port 48: Trunk → DLink│
                  └───────────┬────────────┘
                              │ Uplink (Trunk or Access — see below)
                              ▼
                  ┌────────────────────────┐
                  │  D-Link 8-Port Switch  │
                  │  (Unmanaged Gigabit)   │
                  │                        │
                  │  Ports 1-8: End devices│   (IoT, Guest, or Desk expansion)
                  └────────────────────────┘
```

### 1.2 Role of Each Switch

| Switch | Role | Why |
|--------|------|-----|
| **HPE Aruba 1930 (48-port)** | **Core / Distribution Switch** | Supports VLANs, 802.1Q trunking, QoS, IGMP snooping, port mirroring, and management ACLs. This is the backbone of your network. Every VLAN-aware connection terminates here. |
| **D-Link 8-Port (Unmanaged)** | **Edge / Dumb Extension** | Use for a single-VLAN segment where you need extra ports (e.g., a desk cluster or an IoT closet). Since it's unmanaged, it **cannot** separate VLANs — assign it to **one VLAN only** via the Aruba uplink port as an **access port**. |

> [!IMPORTANT]
> Because the D-Link is unmanaged, **do not trunk multiple VLANs to it**. Set the Aruba uplink port to the D-Link as an **access port** on whichever single VLAN you want those 8 ports to serve (recommended: VLAN 30 — IoT/Guest, or VLAN 20 — Servers, depending on your use case).

### 1.3 VLAN Design

| VLAN ID | Name | Purpose | Subnet | Gateway (Sophos) |
|---------|------|---------|--------|-------------------|
| **1** | Native/Default | Switch management only (restrict access) | `10.0.1.0/24` | `10.0.1.1` |
| **10** | Management | Hypervisor IPMI/iLO, switch management UI, Sophos admin | `10.0.10.0/24` | `10.0.10.1` |
| **20** | Servers | All production VMs (web servers, databases, NAS, Traccar, etc.) | `10.0.20.0/24` | `10.0.20.1` |
| **30** | IoT / Guest | Smart home devices, cameras, guest Wi-Fi | `10.0.30.0/24` | `10.0.30.1` |
| **40** | Trusted LAN | Personal workstations, laptops | `10.0.40.0/24` | `10.0.40.1` |

#### VLAN Firewall Policy Matrix

| Source → Dest | Management (10) | Servers (20) | IoT/Guest (30) | Trusted LAN (40) | WAN |
|---------------|:-:|:-:|:-:|:-:|:-:|
| **Management (10)** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Servers (20)** | ❌ | ✅ | ❌ | ❌ | ✅ (selective) |
| **IoT/Guest (30)** | ❌ | ❌ | ✅ | ❌ | ✅ (filtered) |
| **Trusted LAN (40)** | ✅ (limited) | ✅ | ❌ | ✅ | ✅ |

> [!TIP]
> Only the **Management VLAN** should have full access to everything. IoT/Guest should be completely isolated — internet only, no lateral movement.

---

## 2. Storage Layout Strategy

### 2.1 Drive Allocation

```
┌─────────────────────────────────────────────┐
│              2 TB SSD (Fast I/O)            │
├─────────────────────────────────────────────┤
│  Partition 1:  Hypervisor OS        ~64 GB  │
│  Partition 2:  VM Storage (local)  ~1.9 TB  │
│    ├── Sophos XG VM disk           ~40 GB   │
│    ├── Primary Server VMs          ~500 GB  │
│    ├── Database VMs (MySQL/Postgres)~200 GB  │
│    └── Remaining: hot VM storage   ~1.1 TB  │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│              4 TB HDD (Bulk Storage)        │
├─────────────────────────────────────────────┤
│  Partition 1:  NAS / File Share     ~3.5 TB │
│    ├── Media files                           │
│    ├── Backups (VM snapshots)                │
│    ├── Video storage (dashcam, CCTV)         │
│    └── ISO images & templates                │
│  Partition 2:  VM Backup Target     ~500 GB │
│    └── Scheduled backup dumps                │
└─────────────────────────────────────────────┘
```

### 2.2 Best Practices

| Principle | Implementation |
|-----------|---------------|
| **OS on SSD** | Hypervisor boot partition on SSD for fast startup and responsiveness |
| **Hot VMs on SSD** | All actively-running VMs (Sophos, web servers, databases) get their virtual disks on the SSD for low-latency I/O |
| **Cold storage on HDD** | Backups, media, ISOs, and infrequently-accessed bulk data live on the HDD |
| **Separate backup target** | Dedicate a partition on the HDD exclusively for VM backups — never mix backups with active data |
| **Use LVM / ZFS** | On Proxmox, use **ZFS** on the SSD for snapshots & compression. Use **ext4** or **XFS** on the HDD for straightforward NAS storage |
| **NAS VM** | Create a dedicated NAS VM (TrueNAS Scale or OpenMediaVault) with the HDD passed through via **PCI passthrough** or **raw disk mapping** for best performance |

> [!WARNING]
> With only one copy of each drive, you have **zero redundancy**. Implement an off-site backup strategy (e.g., weekly rsync to a cloud provider or external USB drive rotation) to protect against drive failure.

---

## 3. Virtual Sophos Firewall Configuration

### 3.1 NIC-to-VM Assignment

Your hypervisor host needs **at minimum 2 physical NICs**:

| Physical NIC | vSwitch / Bridge | Sophos vNIC | Purpose |
|-------------|-----------------|-------------|---------|
| **NIC 1** (e.g., `eno1`) | `vmbr0` (WAN bridge, **no IP on host**) | `Port 1 (WAN)` | Receives ISP connection directly. No other VM should be on this bridge. |
| **NIC 2** (e.g., `eno2`) | `vmbr1` (LAN bridge, **trunk to Aruba**) | `Port 2 (LAN)` | Carries all internal VLAN traffic. Other VMs also connect to this bridge. |

#### Proxmox Configuration Example

```bash
# /etc/network/interfaces (on Proxmox host)

# WAN Bridge — dedicated to Sophos WAN
auto vmbr0
iface vmbr0 inet manual
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    # NO IP address here — Sophos handles WAN

# LAN Bridge — trunk carrying all VLANs
auto vmbr1
iface vmbr1 inet manual
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    # Host management IP (VLAN 10)
    # Configure via vmbr1.10 sub-interface if needed
```

#### Sophos VM Hardware Settings (Proxmox)

| Setting | Value |
|---------|-------|
| CPU | 2-4 cores |
| RAM | 4-6 GB |
| Disk | 40 GB (on SSD storage) |
| NIC 1 | `vmbr0` (WAN), Model: `VirtIO` |
| NIC 2 | `vmbr1` (LAN), Model: `VirtIO` |
| Boot | Sophos XG ISO, then configure |

### 3.2 Sophos Initial Configuration Checklist

Once the Sophos VM boots and you access the web admin (default: `https://172.16.16.16:4444`):

#### A. Basic System Setup

1. **Set admin password** — Change from default immediately
2. **Set hostname & timezone** — e.g., `fw01.home.lab`, your local timezone
3. **Register & activate license** — Sophos XG Home Edition (free, up to 4 cores)
4. **Update firmware** — Apply the latest patches before configuring rules

#### B. WAN Interface

1. Navigate to **Network → Interfaces → Port 1**
2. Set **Zone: WAN**
3. Configure IP assignment:
   - **DHCP** if your ISP modem assigns IPs
   - **PPPoE** if your ISP requires it
   - **Static** if you have a fixed IP
4. Enable **Ping** (for testing only — disable later)

#### C. LAN Interfaces (VLAN Sub-Interfaces)

Create a sub-interface for each VLAN on **Port 2**:

| Sub-Interface | VLAN ID | Zone | IP Address | DHCP Server |
|--------------|---------|------|------------|-------------|
| Port2.10 | 10 | LAN | `10.0.10.1/24` | Optional (for mgmt devices) |
| Port2.20 | 20 | DMZ | `10.0.20.1/24` | Yes — pool `10.0.20.100-200` |
| Port2.30 | 30 | Guest* | `10.0.30.1/24` | Yes — pool `10.0.30.100-200` |
| Port2.40 | 40 | LAN | `10.0.40.1/24` | Yes — pool `10.0.40.100-200` |

> *Create a custom zone named "IoT_Guest" under **Network → Zones** for VLAN 30.

#### D. Essential Firewall Rules

Create these rules under **Rules and Policies → Firewall Rules**:

```
Rule 1: Trusted LAN → WAN (Allow All Outbound)
  Source Zone:      LAN (VLAN 40)
  Dest Zone:        WAN
  Action:           ALLOW
  Services:         Any
  NAT:              Masquerade (SNAT)

Rule 2: Servers → WAN (Selective Outbound)
  Source Zone:      DMZ (VLAN 20)
  Dest Zone:        WAN
  Action:           ALLOW
  Services:         HTTP, HTTPS, DNS, NTP, SMTP
  NAT:              Masquerade

Rule 3: IoT/Guest → WAN (Filtered Internet Only)
  Source Zone:      IoT_Guest (VLAN 30)
  Dest Zone:        WAN
  Action:           ALLOW
  Services:         HTTP, HTTPS, DNS
  NAT:              Masquerade
  Web Filter:       Apply "Default" web filter policy

Rule 4: Trusted LAN → Servers (Allow)
  Source Zone:      LAN (VLAN 40)
  Dest Zone:        DMZ (VLAN 20)
  Action:           ALLOW
  Services:         Any

Rule 5: Block All Inter-VLAN (Implicit Deny)
  Source Zone:      Any
  Dest Zone:        Any
  Action:           DROP
  Log:              Enabled
  Position:         LAST (catch-all)

Rule 6: Management Access to Sophos Admin
  Source Zone:      LAN (VLAN 10)
  Dest Zone:        Local (Sophos itself)
  Action:           ALLOW
  Services:         HTTPS (port 4444), SSH (port 22)
```

#### E. Additional Security Hardening

| Setting | Location | Action |
|---------|----------|--------|
| **DNS Protection** | Network Services → DNS | Enable DNS proxy, block known malware domains |
| **IPS** | Protect → Intrusion Prevention | Enable with "LAN to WAN" policy, balanced mode |
| **Web Filtering** | Protect → Web → Policies | Apply content filter to IoT/Guest zone |
| **Admin ACL** | Administration → Device Access | Restrict admin UI access to VLAN 10 only |
| **HTTPS Scanning** | Protect → Web → HTTPS Scanning | Enable for content inspection (optional, add CA cert to clients) |
| **Logging** | System Services → Log Settings | Enable firewall logging, set syslog to a server VM for retention |
| **NTP** | Administration → Time | Sync to `pool.ntp.org` |

---

## 4. Step-by-Step Execution Plan

> [!CAUTION]
> **Follow this order precisely.** Deviating may lock you out of the network or leave your infrastructure exposed. Each phase builds on the previous one.

### Phase 0 — Preparation (Before You Touch Anything)

- [ ] **0.1** Download all required ISOs and save to a USB drive:
  - Proxmox VE ISO (or ESXi)
  - Sophos XG Firewall ISO (register for Home license first)
  - Any other VM ISOs (TrueNAS, Ubuntu Server, etc.)
- [ ] **0.2** Document your ISP settings:
  - Modem model, login credentials
  - WAN IP type (DHCP, PPPoE, static)
  - DNS servers
- [ ] **0.3** Label your physical NICs:
  - Note which port is `NIC 1` (WAN) and which is `NIC 2` (LAN) with masking tape
- [ ] **0.4** Set ISP modem to **bridge mode** (or note its subnet if keeping NAT — double-NAT is not recommended)
- [ ] **0.5** Keep a **separate laptop** connected directly to the ISP modem (or via mobile hotspot) for troubleshooting internet access during setup

### Phase 1 — Install Hypervisor

- [ ] **1.1** Boot the host from the Proxmox USB installer
- [ ] **1.2** During installation:
  - Install OS to the **2 TB SSD** (it will use ~8-15 GB)
  - Set management IP: `10.0.10.2/24` (temporary — on raw NIC 2 or vmbr1)
  - Set gateway: leave blank for now (Sophos will be the gateway)
  - Set DNS: `8.8.8.8` temporarily
- [ ] **1.3** Reboot and verify Proxmox web UI is accessible at `https://10.0.10.2:8006` from a laptop plugged directly into NIC 2 (or via a temporary direct cable)
- [ ] **1.4** Configure storage pools:
  - Create **ZFS pool** on remaining SSD space → name it `ssd-vms`
  - Create **Directory storage** on 4 TB HDD → name it `hdd-bulk`

### Phase 2 — Configure Network Bridges on Hypervisor

- [ ] **2.1** Create `vmbr0` — bridge `NIC 1` (WAN), no IP address
- [ ] **2.2** Create `vmbr1` — bridge `NIC 2` (LAN), enable VLAN-aware
- [ ] **2.3** Assign Proxmox management IP to VLAN 10:
  - Set `vmbr1` IP to `10.0.10.2/24` with VLAN tag 10
  - Or create `vmbr1.10` as a sub-interface
- [ ] **2.4** Apply networking changes and **reconnect** (note: you may need to be on VLAN 10 to reconnect — plug laptop into Aruba on an access port tagged VLAN 10)

### Phase 3 — Deploy Sophos XG Virtual Firewall

- [ ] **3.1** Upload Sophos ISO to Proxmox storage
- [ ] **3.2** Create the Sophos VM:
  - 4 cores, 4-6 GB RAM, 40 GB disk on `ssd-vms`
  - NIC 1 → `vmbr0` (WAN)
  - NIC 2 → `vmbr1` (LAN)
  - Both NICs: VirtIO model
- [ ] **3.3** Boot VM, complete Sophos CLI-based initial setup:
  - Default access: `https://172.16.16.16:4444` (connect a temp NIC to the LAN bridge)
  - Set admin password
  - Assign Port 1 → WAN zone, DHCP/PPPoE from ISP
  - Assign Port 2 → LAN zone, IP `10.0.10.1/24`
- [ ] **3.4** Verify internet access through Sophos:
  - From Sophos CLI: `ping 8.8.8.8`
  - From Proxmox host: set gateway to `10.0.10.1`, then `ping 8.8.8.8`

> [!TIP]
> At this point, your Sophos is acting as your entire network's firewall and router. All traffic should flow: Devices → Aruba → NIC 2 → Sophos VM → NIC 1 → ISP Modem.

### Phase 4 — Configure VLANs on Sophos

- [ ] **4.1** Create VLAN sub-interfaces on Sophos Port 2 (as per Section 3.2C)
- [ ] **4.2** Enable DHCP servers on each VLAN subnet
- [ ] **4.3** Create the firewall rules (as per Section 3.2D)
- [ ] **4.4** Test: plug a device into a VLAN 40 access port → confirm DHCP lease in `10.0.40.x` range and internet access

### Phase 5 — Configure HPE Aruba 1930 Switch

- [ ] **5.1** Factory reset the Aruba switch if previously used
- [ ] **5.2** Connect to its default management IP (check manual — often `192.168.1.1` or via Aruba Instant On app)
- [ ] **5.3** Create VLANs:
  - VLAN 10, 20, 30, 40 with matching names
- [ ] **5.4** Configure port assignments:

| Port(s) | Mode | VLAN(s) | Purpose |
|---------|------|---------|---------|
| 1 | **Trunk** | 10, 20, 30, 40 (tagged), 1 (native) | Uplink to Hypervisor NIC 2 |
| 2–8 | Access | 10 (untagged) | Management devices |
| 9–24 | Access | 20 (untagged) | Server NICs, if any physical servers |
| 25–40 | Access | 40 (untagged) | Trusted workstations, PCs |
| 41–47 | Access | 30 (untagged) | IoT devices, cameras |
| 48 | **Access** | 30 (untagged) | Uplink to D-Link (IoT segment) |

- [ ] **5.5** Set switch management VLAN to **VLAN 10**, IP `10.0.10.3/24`, gateway `10.0.10.1`
- [ ] **5.6** Verify switch management UI is accessible from a VLAN 10 device

### Phase 6 — Connect D-Link Switch

- [ ] **6.1** Cable D-Link uplink to Aruba **Port 48** (configured as access port on VLAN 30)
- [ ] **6.2** All 8 D-Link ports now operate on VLAN 30 (IoT/Guest)
- [ ] **6.3** Test: plug an IoT device into the D-Link → verify it gets `10.0.30.x` DHCP → confirm internet access but **no access** to `10.0.20.x` or `10.0.40.x` subnets

### Phase 7 — Deploy Remaining VMs

- [ ] **7.1** Create a **NAS VM** (TrueNAS Scale or OpenMediaVault):
  - Pass through the 4 TB HDD via raw disk mapping
  - Network: `vmbr1`, VLAN tag 20, IP `10.0.20.10`
  - Configure SMB/NFS shares
- [ ] **7.2** Create additional server VMs as needed:
  - Assign all to `vmbr1` with VLAN tag 20
  - Static IPs in `10.0.20.x` range
- [ ] **7.3** Create a backup schedule:
  - Proxmox Backup: snapshot VMs nightly to `hdd-bulk` storage
  - NAS Backup: rsync critical data off-site weekly

### Phase 8 — Security Hardening & Validation

- [ ] **8.1** Restrict Proxmox admin UI access to VLAN 10 only (host firewall rules)
- [ ] **8.2** Restrict Sophos admin UI to VLAN 10 only (Device Access settings)
- [ ] **8.3** Restrict Aruba switch management to VLAN 10 only
- [ ] **8.4** Disable Sophos WAN ping response (hardening)
- [ ] **8.5** Enable Sophos IPS on LAN-to-WAN traffic
- [ ] **8.6** Run validation tests:

| Test | Expected Result |
|------|----------------|
| VLAN 40 device → Internet | ✅ Works |
| VLAN 40 device → VLAN 20 server | ✅ Works |
| VLAN 30 device → Internet | ✅ Works (filtered) |
| VLAN 30 device → VLAN 20 server | ❌ Blocked |
| VLAN 30 device → VLAN 40 device | ❌ Blocked |
| VLAN 20 server → VLAN 40 | ❌ Blocked |
| External port scan on WAN IP | ❌ All ports stealth/closed |

- [ ] **8.7** Document all IPs, VLANs, passwords in a secure password manager (e.g., Bitwarden)

---

## Quick Reference — IP Address Map

| Device | VLAN | IP Address | Purpose |
|--------|------|------------|---------|
| Sophos WAN | — | DHCP from ISP | Internet gateway |
| Sophos VLAN 10 GW | 10 | `10.0.10.1` | Management gateway |
| Sophos VLAN 20 GW | 20 | `10.0.20.1` | Server gateway |
| Sophos VLAN 30 GW | 30 | `10.0.30.1` | IoT/Guest gateway |
| Sophos VLAN 40 GW | 40 | `10.0.40.1` | Trusted LAN gateway |
| Proxmox Host | 10 | `10.0.10.2` | Hypervisor management |
| Aruba Switch | 10 | `10.0.10.3` | Switch management |
| NAS VM | 20 | `10.0.20.10` | File storage |
| D-Link Switch | 30 | N/A (unmanaged) | IoT edge expansion |

---

> [!NOTE]
> **Future Enhancements to Consider:**
> - Add a second SSD for ZFS mirror (redundancy)
> - Deploy a reverse proxy VM (Nginx Proxy Manager) on VLAN 20 for exposing services securely
> - Set up WireGuard VPN on Sophos for remote access to VLAN 10/20
> - Add a Pi-hole or AdGuard Home VM on VLAN 20 as your network-wide DNS filter
> - Implement 802.1X port authentication on the Aruba switch for dynamic VLAN assignment
