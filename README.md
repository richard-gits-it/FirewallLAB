# ITEC285 — Lab 04: Firewalls & OPNsense

**Course:** ITEC285 — Penetration Testing  
**Lab:** 04 — Firewalls and Proxies  
**Student:** RJ Lorenzo  
**Date:** January 28, 2026

---

## Overview

This lab involved deploying and configuring an **OPNsense** firewall (BSD-based, v25.7) in a simulated network environment using Hyper-V. The setup created a segmented LAN/WAN topology to explore core firewall concepts: interface assignment, ICMP rules, NAT port forwarding, and DHCP configuration via Kea DHCPv4.

---

## Network Topology

```
[Kali Linux]          [OPNsense Firewall]          [Windows 11 / IIS]
172.18.28.x  ───WAN──► hn1 (Red/External)
(College Net)         hn0 (Green/Internal) ◄──LAN── 192.168.1.100/24
                      192.168.1.1
```

| Host | Interface | IP Address |
|------|-----------|------------|
| OPNsense WAN | Red (College Net) | 172.18.28.213 (DHCP) |
| OPNsense LAN | Green (Internal) | 192.168.1.1 |
| Windows 11 | Green (Internal) | 192.168.1.100 (via DHCP) |
| Kali Linux | College Network | 172.18.x.x |

---

## Lab Tasks Completed

### 1. Environment Setup
- Created Hyper-V **"Green Network"** internal virtual switch to simulate a LAN segment
- Assigned Windows 11 VM to Green Network; Kali remained on the external college network
- Installed **OPNsense 25.7** in a Gen 2 Hyper-V VM (2GB RAM, 20GB disk, Secure Boot disabled)
- Assigned `BE:EF:DE:AD:BE:EF` MAC to the Green NIC for deterministic interface assignment
- Completed OPNsense setup wizard: hostname `OPNsense-Richard`, timezone `America/Edmonton`, WAN via DHCP, RFC1918 blocks removed for lab purposes

### 2. Windows Firewall Behavior (Pre-OPNsense)
Ran `nmap` against the Windows 11 VM from Kali with firewall **on** vs **off**:

| Firewall State | Open Ports Found |
|----------------|-----------------|
| Enabled | 4 ports |
| Disabled | 1 port (80/HTTP) |

**Observation:** The Windows Firewall was actively blocking ports — disabling it exposed fewer services because IIS on port 80 was already accessible; the firewall was blocking other ports that were visible only when it was active (e.g., filtered → open transitions on service ports).

### 3. ICMP Rule — Allow Ping on WAN
Added a Firewall → Rules → WAN rule to permit ICMPv4 Echo Request/Reply on the external interface (for lab troubleshooting purposes only — not a production best practice).

- **Result:** Kali successfully pinged OPNsense WAN (172.18.28.213) ✅
- **Result:** Windows 11 successfully pinged OPNsense LAN (192.168.1.1) ✅

**Nmap of OPNsense external IP:**
```
$ nmap 172.18.28.213
# Result: 0 open ports
```
> **Why 0 ports?** OPNsense enforces a default-deny posture — no services are exposed on the WAN unless explicitly permitted. The ICMP rule allows pings but doesn't open any TCP/UDP ports.

### 4. NAT Port Forwarding — Expose IIS Externally
Configured Firewall → NAT → Port Forward to redirect inbound TCP port 80 on the WAN interface to the Windows 11 internal IP (`192.168.1.100:80`).

- Selected **"Add associated filter rule"** to auto-create the matching firewall allow rule
- Logged the rule; categorized as `Lab`
- **Result:** Kali browsed to `http://172.18.28.213` and reached the IIS welcome page ✅

### 5. Kea DHCPv4 Configuration
Enabled and configured the OPNsense Kea DHCP server for the LAN segment:

| Setting | Value |
|---------|-------|
| Interface | LAN |
| Subnet | 192.168.1.0/24 |
| Pool | 192.168.1.100 – 192.168.1.200 |
| Domain | itec285.local |

**DHCP Reservation:**
| Field | Value |
|-------|-------|
| MAC | `AB:CD:EF:AB:CD:EF` |
| Reserved IP | 192.168.1.181 |
| Hostname | Richard-Laptop |

Windows 11 was switched to DHCP and received `192.168.1.100` from the pool. ✅

---

## Screenshots

| File | Description |
|------|-------------|
| [`ping-internal.png`](screenshots/ping-internal.png) | Kali pinging OPNsense WAN (172.18.28.213) |
| [`ping-external.png`](screenshots/ping-external.png) | Windows 11 pinging OPNsense LAN (192.168.1.1) |
| [`External-http.png`](screenshots/External-http.png) | Kali accessing IIS via OPNsense WAN + NAT port forward |
| [`DHCP-Leases.png`](screenshots/DHCP-Leases.png) | Kea DHCPv4 active lease for Windows 11 |
| [`DHCP-Reservations.png`](screenshots/DHCP-Reservations.png) | Kea DHCPv4 static reservation (Richard-Laptop) |

---

## Key Concepts Demonstrated

- **Default-deny firewall policy** — OPNsense blocks all inbound WAN traffic unless explicitly permitted
- **Stateful inspection** — outbound traffic and return traffic are tracked automatically
- **NAT port forwarding** — translates external connections to internal hosts (PAT/DNAT)
- **DHCP scope + reservations** — dynamic address assignment with MAC-based static leases
- **Network segmentation** — LAN/WAN separation enforced at the hypervisor and firewall level
- **Defense in depth** — host-based firewall (Windows) + network firewall (OPNsense) layered

---

## Tools & Technologies

- **OPNsense 25.7** (FreeBSD-based firewall/router)
- **Kea DHCPv4** (modern DHCP server bundled with OPNsense)
- **Hyper-V** (virtualization, virtual switch management)
- **Nmap** (port scanning for firewall verification)
- **Kali Linux** (attacker/external perspective)
- **Windows 11 + IIS** (target/internal web server)
