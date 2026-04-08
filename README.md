# Multi-VLAN Homelab Network Segmentation
### ASUS GT-AX11000 + Asuswrt-Merlin | Proxmox Cybersecurity Lab

---

> **Note on IP Addressing:** All IP addresses and subnets in this document have been replaced with descriptive placeholders for operational security reasons. Substitute your own RFC 1918 address space when implementing. Common choices are the 10.0.0.0/8 or 192.168.0.0/16 ranges.

---

## Project Overview

This project documents the design and implementation of a fully segmented homelab network using VLANs on a consumer router running custom firmware. The goal was to create isolated network zones for a cybersecurity learning environment — separating a daily-driver management network, a Proxmox hypervisor, and multiple isolated lab environments, each with tailored firewall rules enforcing a least-privilege access model.

**Skills demonstrated:**
- Network segmentation design
- Linux bridge and VLAN interface configuration
- iptables firewall rule authoring
- Trunk port and 802.1Q VLAN tagging
- Asuswrt-Merlin persistent scripting
- DHCP scope configuration per VLAN
- Threat modelling and network isolation for a cybersecurity lab

---

## Hardware & Software

| Component | Details |
|-----------|---------|
| Router | ASUS GT-AX11000 |
| Firmware | Asuswrt-Merlin (third-party, based on official ASUS firmware) |
| Hypervisor | Proxmox VE |
| Server NIC | Connected to router LAN Port 2 (eth3) as 802.1Q trunk |
| Management device | Connected via Wi-Fi + dedicated 2.5G LAN port (eth6) |

---

## Network Design

### Why VLANs?

A flat network (no segmentation) means every device can communicate with every other device by default. In a cybersecurity lab environment this is dangerous — attack tools, vulnerable machines, and malware samples share the same network as trusted daily-use devices.

VLANs (Virtual Local Area Networks) solve this by creating logically separated broadcast domains on the same physical infrastructure. Each VLAN behaves as its own independent network. Traffic between VLANs can only flow if explicitly permitted by firewall rules, enforcing a **zero-trust perimeter** between zones.

### VLAN Architecture

```
Internet
    │
    │ (WAN)
┌───┴────────────────────────────────────────────┐
│              ASUS GT-AX11000                    │
│                                                 │
│  br0           br20          br40        br50   │
│  VLAN 10       VLAN 20       VLAN 40     VLAN 50│
│  <MGMT_GW>     <PROXMOX_GW>  <WEB_GW>   <IOT_GW>│
│                                                 │
│  eth6(2.5G)    eth3 (trunk port ─────────────┐  │
│  eth2,4,5      tagged: 20, 40, 50            │  │
└─────────────────────────────────────────────-┼──┘
                                               │
                              ┌────────────────┴──────────────┐
                              │          Proxmox Server        │
                              │                                │
                              │  eno1.20 → vmbr0 (VLAN 20)    │
                              │  eno1.40 → vmbr3 (VLAN 40)    │
                              │  eno1.50 → vmbr5 (VLAN 50)    │
                              │  vmbr1/2  → VLAN 30            │
                              │           (internal only)      │
                              └────────────────────────────────┘
```

### VLAN Definitions

| VLAN | Name | Subnet | Gateway | Purpose |
|------|------|--------|---------|---------|
| 10 | Management | `<MGMT_SUBNET>` | `<MGMT_GW>` | Daily driver devices, router admin |
| 20 | Proxmox | `<PROXMOX_SUBNET>` | `<PROXMOX_GW>` | Proxmox hypervisor management GUI |
| 30 | Isolated Lab | `<LAB_SUBNET>` | None | Air-gapped attack lab, Proxmox internal only |
| 40 | Web Lab | `<WEBLAB_SUBNET>` | `<WEB_GW>` | Web-facing lab VMs, limited internet |
| 50 | Cloud/IoT Lab | `<IOT_SUBNET>` | `<IOT_GW>` | IoT simulation, HTTPS-only internet |

### Access Control Matrix

| Source → Destination | VLAN 10 | VLAN 20 | VLAN 30 | VLAN 40 | VLAN 50 | Internet |
|----------------------|---------|---------|---------|---------|---------|----------|
| VLAN 10 (Management) | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| VLAN 20 (Proxmox) | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| VLAN 30 (Isolated Lab) | ❌ | ❌ | Internal | ❌ | ❌ | ❌ |
| VLAN 40 (Web Lab) | ❌ | ❌ | ❌ | ✅ | ❌ | HTTP/HTTPS only |
| VLAN 50 (Cloud/IoT) | ❌ | ❌ | ❌ | ❌ | ✅ | HTTPS only |

---

## Technical Implementation

### Why Asuswrt-Merlin?

The stock ASUS firmware on the GT-AX11000 exposes only basic switch controls (jumbo frames, link aggregation) with no VLAN configuration UI. Asuswrt-Merlin is a community-maintained firmware built directly on top of ASUS's own codebase. It is not a jailbreak — it preserves all stock functionality while adding:

- JFFS2 persistent partition for custom scripts
- SSH access
- Script hooks that run at defined points in the boot process (`init-start`, `firewall-start`, etc.)

### Key Concepts

#### 802.1Q VLAN Tagging
IEEE 802.1Q is the standard that allows multiple VLANs to share a single physical link. Each Ethernet frame is tagged with a 4-byte header containing the VLAN ID (1–4094). Devices that understand these tags (switches, servers, hypervisors) read the tag and forward the frame to the correct VLAN. Devices that don't understand tags (regular PCs) receive untagged frames on their access port.

#### Trunk Ports vs Access Ports
- **Access port** — carries a single untagged VLAN. Regular devices plug in here.
- **Trunk port** — carries multiple tagged VLANs on one cable. Servers and managed switches connect here.

LAN Port 2 (eth3) on this router is configured as a trunk port carrying VLANs 20, 40, and 50 tagged. Proxmox receives all three as tagged frames and routes them to the appropriate virtual bridge.

#### Linux Bridges
Linux uses software bridges (`brctl`) as virtual switches. Each VLAN gets its own bridge (`br20`, `br40`, `br50`). VLAN sub-interfaces on the physical port (`eth3.20`, `eth3.40`, `eth3.50`) are attached to their respective bridges. The bridge's assigned IP address becomes the default gateway for that VLAN.

The existing `br0` serves as the VLAN 10 (Management) bridge and is left intact — only `eth3` is removed from it so it no longer carries the trunk port.

#### iptables and the FORWARD Chain
iptables is the Linux kernel firewall. The `FORWARD` chain processes packets passing **through** the router (inter-VLAN and internet-bound traffic). A custom chain `VLANS` is inserted at position 1 of FORWARD so it evaluates before any default rules.

Rule ordering is critical:
1. `ESTABLISHED,RELATED` — allows return traffic for existing connections first
2. Specific ALLOW rules for permitted cross-VLAN traffic
3. DROP rules blocking all other inter-VLAN traffic
4. NAT MASQUERADE rules for internet access

---

## Script Files

### File 1: `/jffs/scripts/init-start`
Runs at boot before networking is fully initialized. Responsible for:
- Removing the trunk port (eth3) from the default LAN bridge (br0)
- Creating 802.1Q VLAN sub-interfaces on eth3
- Creating Linux bridges for each VLAN
- Assigning gateway IP addresses to each bridge

```bash
#!/bin/sh
# ============================================================
# VLAN INIT - GT-AX11000 | Asuswrt-Merlin
# ============================================================
# Replace all <PLACEHOLDER> values with your chosen subnets
# ============================================================

# Remove eth3 from the default LAN bridge
ip link set eth3 down
brctl delif br0 eth3
ip link set eth3 up

# Create tagged VLAN sub-interfaces on eth3
ip link add link eth3 name eth3.20 type vlan id 20
ip link add link eth3 name eth3.40 type vlan id 40
ip link add link eth3 name eth3.50 type vlan id 50

ip link set eth3.20 up
ip link set eth3.40 up
ip link set eth3.50 up

# Create bridges for each VLAN
brctl addbr br20
brctl addbr br40
brctl addbr br50

brctl addif br20 eth3.20
brctl addif br40 eth3.40
brctl addif br50 eth3.50

ip link set br20 up
ip link set br40 up
ip link set br50 up

# Assign router gateway IPs — replace with your chosen addresses
ip addr add <PROXMOX_GW>/24 dev br20
ip addr add <WEB_GW>/24 dev br40
ip addr add <IOT_GW>/24 dev br50
```

### File 2: `/jffs/configs/dnsmasq.conf.add`
Extends the router's built-in dnsmasq DHCP server to serve addresses on each new VLAN subnet. VLAN 30 has no entry here because it has no router-side gateway — it is entirely internal to Proxmox.

```
# VLAN 20 - Proxmox
interface=br20
dhcp-range=br20,<PROXMOX_DHCP_START>,<PROXMOX_DHCP_END>,255.255.255.0,12h

# VLAN 40 - Web Lab
interface=br40
dhcp-range=br40,<WEBLAB_DHCP_START>,<WEBLAB_DHCP_END>,255.255.255.0,12h

# VLAN 50 - Cloud/IoT Lab
interface=br50
dhcp-range=br50,<IOT_DHCP_START>,<IOT_DHCP_END>,255.255.255.0,12h

# VLAN 30 - No DHCP, Proxmox internal only
```

> **DHCP range design:** Reserve .1–.99 for static assignments (gateway is always .1, servers get static IPs in this range). Set DHCP to hand out .100 and above to keep addressing predictable.

### File 3: `/jffs/scripts/firewall-start`
Runs after the firewall is initialized on each boot. Inserts all inter-VLAN firewall rules into a dedicated `VLANS` chain.

```bash
#!/bin/sh
# ============================================================
# FIREWALL RULES - Multi-VLAN Setup
# GT-AX11000 | Asuswrt-Merlin
# Replace all <PLACEHOLDER> values with your chosen subnets
# ============================================================

# Create and flush custom chain
iptables -N VLANS 2>/dev/null
iptables -F VLANS
iptables -I FORWARD 1 -j VLANS

# Allow return traffic for established connections (must be first)
iptables -A VLANS -m state --state ESTABLISHED,RELATED -j ACCEPT

# VLAN 10 - Management
# Allow → Proxmox GUI | Block → Isolated Lab
iptables -A VLANS -i br0 -o br20 -j ACCEPT
iptables -A VLANS -i br0 -d <LAB_SUBNET> -j DROP

# VLAN 20 - Proxmox
# Block all inter-VLAN outbound | Allow internet | Block all inbound
iptables -A VLANS -i br20 -d <MGMT_SUBNET>   -j DROP
iptables -A VLANS -i br20 -d <LAB_SUBNET>    -j DROP
iptables -A VLANS -i br20 -d <WEBLAB_SUBNET> -j DROP
iptables -A VLANS -i br20 -d <IOT_SUBNET>    -j DROP
iptables -A VLANS -i br20 -j ACCEPT
iptables -A VLANS -o br20 -j DROP
iptables -t nat -A POSTROUTING -s <PROXMOX_SUBNET> -j MASQUERADE

# VLAN 30 - Isolated Lab (hard block at perimeter — defence in depth)
iptables -A VLANS -s <LAB_SUBNET> -j DROP
iptables -A VLANS -d <LAB_SUBNET> -j DROP

# VLAN 40 - Web Lab
# Block all inter-VLAN | Allow HTTP + HTTPS outbound only | Block all inbound
iptables -A VLANS -i br40 -d <MGMT_SUBNET>    -j DROP
iptables -A VLANS -i br40 -d <PROXMOX_SUBNET> -j DROP
iptables -A VLANS -i br40 -d <LAB_SUBNET>     -j DROP
iptables -A VLANS -i br40 -d <IOT_SUBNET>     -j DROP
iptables -A VLANS -i br40 -p tcp --dport 80   -j ACCEPT
iptables -A VLANS -i br40 -p tcp --dport 443  -j ACCEPT
iptables -A VLANS -i br40 -j DROP
iptables -A VLANS -o br40 -j DROP
iptables -t nat -A POSTROUTING -s <WEBLAB_SUBNET> -j MASQUERADE

# VLAN 50 - Cloud/IoT Lab
# Block all inter-VLAN | Allow HTTPS only | Block all inbound
iptables -A VLANS -i br50 -d <MGMT_SUBNET>    -j DROP
iptables -A VLANS -i br50 -d <PROXMOX_SUBNET> -j DROP
iptables -A VLANS -i br50 -d <LAB_SUBNET>     -j DROP
iptables -A VLANS -i br50 -d <WEBLAB_SUBNET>  -j DROP
iptables -A VLANS -i br50 -p tcp --dport 443  -j ACCEPT
iptables -A VLANS -i br50 -j DROP
iptables -A VLANS -o br50 -j DROP
iptables -t nat -A POSTROUTING -s <IOT_SUBNET> -j MASQUERADE
```

---

## Deployment

### Prerequisites
1. Asuswrt-Merlin firmware flashed
2. In router GUI: **Administration → System → Enable JFFS custom scripts** → Apply
3. In router GUI: **Administration → System → Enable SSH** → Apply

### Applying the Scripts

```bash
# Set execute permissions on script files
chmod a+rx /jffs/scripts/init-start
chmod a+rx /jffs/scripts/firewall-start

# Apply live without rebooting
sh /jffs/scripts/init-start
service restart_dnsmasq
sh /jffs/scripts/firewall-start
```

Scripts in `/jffs/scripts/` are automatically executed by Merlin on each boot, so the configuration is fully persistent. After applying live, reboot once to confirm everything comes up automatically.

---

## Verification

### Router-Side Checks

```bash
# Verify bridges exist and have correct IPs
ip addr show br20
ip addr show br40
ip addr show br50

# Verify eth3 is no longer in the default LAN bridge
brctl show br0

# Verify VLAN sub-interfaces are UP
ip link show | grep eth3

# Verify DHCP is listening on new interfaces
netstat -ulnp | grep 67

# Verify firewall rules loaded correctly
iptables -L VLANS --line-numbers
```

### Network Behaviour Tests

**From VLAN 20 (Proxmox):**
```bash
ping <PROXMOX_GW>    # ✅ Gateway reachable
ping <MGMT_GW>       # ❌ VLAN 10 blocked
curl https://google.com  # ✅ Internet allowed
```

**From VLAN 40 (Web Lab):**
```bash
curl http://example.com   # ✅ HTTP allowed
curl https://example.com  # ✅ HTTPS allowed
ping 8.8.8.8              # ❌ ICMP blocked
ping <PROXMOX_GW>         # ❌ Inter-VLAN blocked
```

**From VLAN 50 (Cloud/IoT):**
```bash
curl https://google.com  # ✅ HTTPS allowed
curl http://example.com  # ❌ HTTP blocked
ping <MGMT_GW>           # ❌ Inter-VLAN blocked
```

---

## Security Design Rationale

**VLAN 30 — Defence in Depth**
VLAN 30 uses two independent layers of isolation. The primary layer is Proxmox itself — `vmbr1` and `vmbr2` have no physical port assigned, so traffic can never leave the hypervisor regardless of VM behaviour. The secondary layer is iptables hard-drop rules on the router as a fallback. No single point of failure can break isolation.

**VLAN 40 — Port-Restricted Internet**
Only TCP ports 80 and 443 are permitted outbound. ICMP, UDP, and all other TCP ports are blocked. This limits the utility of the network for C2 callbacks over non-standard ports while still allowing package installation and web research.

**VLAN 50 — HTTPS-Only**
IoT and cloud lab devices are restricted to TCP 443 only, mirroring how enterprise security teams lock down IoT segments. Anything attempting to communicate over a non-HTTPS channel is exhibiting suspicious behaviour.

**Custom iptables Chain**
Rather than inserting rules directly into FORWARD, a dedicated `VLANS` chain is used. This makes the ruleset modular, auditable, and prevents conflicts with Merlin's default rules. The chain is flushed and rebuilt on each boot to avoid rule duplication.

---

## Proxmox NIC Configuration

The Proxmox server's physical NIC connected to the router trunk port must be configured to process 802.1Q tags. In `/etc/network/interfaces` on Proxmox:

```
# Physical NIC as trunk — no IP assigned here
auto eno1
iface eno1 inet manual

# VLAN 20 sub-interface → Proxmox management bridge
auto eno1.20
iface eno1.20 inet manual

auto vmbr0
iface vmbr0 inet static
    address <PROXMOX_IP>/24
    gateway <PROXMOX_GW>
    bridge-ports eno1.20
    bridge-stp off
    bridge-fd 0

# VLAN 40 → Web Lab bridge
auto eno1.40
iface eno1.40 inet manual

auto vmbr3
iface vmbr3 inet manual
    bridge-ports eno1.40
    bridge-stp off
    bridge-fd 0

# VLAN 50 → Cloud/IoT bridge
auto eno1.50
iface eno1.50 inet manual

auto vmbr5
iface vmbr5 inet manual
    bridge-ports eno1.50
    bridge-stp off
    bridge-fd 0

# VLAN 30 → Isolated lab (no physical port — air-gapped by architecture)
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

> Replace `eno1` with your actual NIC name (`ip link show` to find it). Set `<PROXMOX_IP>` to a static address outside your DHCP range (e.g. .10) so it never changes.

---

## Lessons Learned

**Stock firmware limitations** — The GT-AX11000 stock firmware advertises switch control features that in practice only expose link aggregation and jumbo frames. Even after flashing Merlin, the VLAN GUI tab is absent on this model. Full VLAN support requires scripting via SSH.

**robocfg vs ip/brctl** — Many guides reference `robocfg` for ASUS VLAN configuration. This tool was not present on this unit. Using `ip link` with VLAN type sub-interfaces and `brctl` bridges achieves the same result using standard Linux tooling, regardless of chipset.

**iptables rule order is a security property** — The `ESTABLISHED,RELATED` rule must come first. An incorrect order causes legitimate traffic to drop or blocked traffic to pass. Every rule's position is a deliberate security decision.

**Architecture beats firewall rules** — VLAN 30's strongest protection is the absence of a physical bridge port in Proxmox, not the iptables rules. An absent interface cannot forward traffic. Rules can have bugs.

**Verify hardware before writing config** — Port numbering on this router does not match interface numbering. Physical cable testing (`ip link show` while plugging/unplugging) confirmed the correct mapping before any configuration was written.

---

## References

- [Asuswrt-Merlin Project](https://www.asuswrt-merlin.net)
- [Asuswrt-Merlin Wiki — User Scripts](https://github.com/RMerl/asuswrt-merlin.ng/wiki/User-scripts)
- [IEEE 802.1Q Standard](https://en.wikipedia.org/wiki/IEEE_802.1Q)
- [Linux Bridge Administration](https://wiki.linuxfoundation.org/networking/bridge)
- [Proxmox Network Configuration](https://pve.proxmox.com/wiki/Network_Configuration)
- [iptables Documentation](https://netfilter.org/documentation/)
