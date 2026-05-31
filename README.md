# 🏥 Melbourne Health Services — Enterprise Network Design
> Cisco Packet Tracer · Hierarchical Network · Dual-Site Healthcare Provider

![Cisco Packet Tracer](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=flat&logo=cisco&logoColor=white)
![Protocol](https://img.shields.io/badge/Routing-OSPF-green?style=flat)
![VPN](https://img.shields.io/badge/VPN-IPSec%20Site--to--Site-orange?style=flat)
![Security](https://img.shields.io/badge/Security-ACL%20%7C%20PAT%20%7C%20Port%20Security-red?style=flat)

---

## 📋 Project Overview

A full enterprise network design and implementation for **Melbourne Health Services**, a dual-site healthcare provider operating a main HQ hospital and a branch hospital located 20km apart. The network was built from scratch to replace third-party IT services with an owned, secure, and cost-effective infrastructure — following the **CIA triad** (Confidentiality, Integrity, Availability).

The design uses a **three-tier hierarchical model** with redundancy, VLAN segmentation per department, dynamic routing, centralised DHCP, wireless access, secure remote management, and an encrypted VPN tunnel between sites.

---

## 🏗️ Network Architecture

### Sites & Departments

| Site | Departments | Users per Dept |
|------|-------------|----------------|
| **HQ (Main Hospital)** | MLOCS, MER, MRM, IT, CS, GWA | ~60 |
| **Branch Hospital** | NSO, HL, HR, MK, FIN, GWA | ~30 |
| **Server Room (HQ)** | DHCP, DNS, Web, Email Servers | Static IPs |

### Physical Topology
- **1 Core Router** per site (HQ Router + Branch Router)
- **2 Multilayer Switches (L3)** per site for redundancy and inter-VLAN routing
- **Access Switches** per department (L2)
- **Cisco Access Points** in every department for wireless
- HQ ↔ Branch connected via **serial WAN link**
- Both routers connect to **2 ISPs** each for redundancy

---

## 🌐 VLAN & IP Addressing

Base Network: `192.168.100.0`

### HQ Departments — /26 subnets (62 usable hosts)

| Department | VLAN | Subnet | Gateway |
|------------|------|--------|---------|
| MLOCS | 10 | 192.168.100.0/26 | 192.168.100.1 |
| MER | 20 | 192.168.100.64/26 | 192.168.100.65 |
| MRM | 30 | 192.168.100.128/26 | 192.168.100.129 |
| IT | 40 | 192.168.100.192/26 | 192.168.100.193 |
| CS | 50 | 192.168.101.0/26 | 192.168.101.1 |
| GWA | 60 | 192.168.101.64/26 | 192.168.101.65 |

### Branch Departments — /27 subnets (30 usable hosts)

| Department | VLAN | Subnet | Gateway |
|------------|------|--------|---------|
| NSO | 80 | 192.168.101.128/27 | 192.168.101.129 |
| HL | 90 | 192.168.101.160/27 | 192.168.101.161 |
| HR | 100 | 192.168.101.192/27 | 192.168.101.193 |
| MK | 110 | 192.168.101.224/27 | 192.168.101.225 |
| FIN | 120 | 192.168.102.0/27 | 192.168.102.1 |
| GWA | 130 | 192.168.102.32/27 | 192.168.102.33 |

### Server Room — VLAN 70

| Network | Subnet | Addressing |
|---------|--------|------------|
| Server Room | 192.168.102.64/28 | **Static** |

Hosts: DHCP Server, DNS Server, Web Server, Email Server

### WAN — Public IP Addressing

| Subnet | Link |
|--------|------|
| 195.136.17.0/30 | HQ Router ↔ ISP1 |
| 195.136.17.4/30 | HQ Router ↔ ISP2 |
| 195.136.17.8/30 | Branch Router ↔ ISP1 |
| 195.136.17.12/30 | Branch Router ↔ ISP2 |

---

## ⚙️ Technologies Implemented

### 1. Basic Device Settings
- Hostnames, console password, enable secret password, banner MOTD
- `no ip domain-lookup` on all devices

### 2. VLANs & Switching
- VLANs created and assigned on all access and multilayer switches
- Access ports assigned to correct VLANs per department
- Trunk ports configured between L2 access switches and L3 multilayer switches

### 3. Inter-VLAN Routing (L3 Switches)
- Switch Virtual Interfaces (SVIs) configured on multilayer switches
- `ip routing` enabled on L3 switches
- `ip helper-address` configured on SVIs to relay DHCP requests to the server room

### 4. DHCP Server
- Centralised dedicated DHCP server in the server room
- Separate pools for all 12 department VLANs
- Server room devices assigned static IPs manually

### 5. OSPF Routing
- OSPF process configured on all routers and L3 switches
- All internal subnets advertised into OSPF area 0
- Default static routes configured using next-hop IPs for internet-bound traffic

### 6. SSH Remote Access
- RSA keys generated on all routers and L3 switches
- SSH version 2 enforced
- VTY lines restricted to SSH only (`transport input ssh`)

### 7. Wireless Network
- Cisco Access Points deployed in every department
- SSIDs bound to respective VLANs

### 8. Port Security (Server Room Switch)
- Maximum 1 MAC address per port
- Sticky MAC address learning
- Violation mode: `shutdown`

### 9. Site-to-Site IPSec VPN
- IKE Phase 1: AES-256 encryption, SHA hashing, pre-shared key authentication
- IKE Phase 2: ESP-AES + ESP-SHA-HMAC transform set
- Crypto map applied on the HQ and Branch router serial interfaces
- Extended ACL defines "interesting traffic" (HQ LAN ↔ Branch LAN)

### 10. PAT (Port Address Translation)
- NAT overload configured on outbound router interfaces
- Internal subnets translated to the public interface IP
- ACL applied to define traffic eligible for PAT

### 11. Access Control Lists
- Extended ACLs used for VPN interesting traffic
- Standard/Extended ACLs enforcing access policies between departments and sites

---

## 🔑 Configuration Reference

```bash
# Basic hardening — all devices
hostname <DeviceName>
no ip domain-lookup
enable secret <password>
line console 0
 password <password>
 login
banner motd # Authorised Access Only #

# SSH setup
ip domain-name mhs.local
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh

# OSPF
router ospf 10
 network 192.168.100.0 0.0.0.63 area 0
 ! ... all subnets

# Default static route
ip route 0.0.0.0 0.0.0.0 <next-hop-IP>

# Port security — server switch
interface fa0/1
 switchport mode access
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown

# IPSec VPN — HQ side
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
crypto isakmp key vpnkey address <Branch-Serial-IP>
crypto ipsec transform-set MYSET esp-aes esp-sha-hmac
crypto map CMAP 10 ipsec-isakmp
 set peer <Branch-Serial-IP>
 set transform-set MYSET
 match address VPN-ACL

# PAT
ip nat inside source list NAT-ACL interface <outbound-interface> overload
```

---

## ✅ Testing & Verification

| Test | Method | Expected Result |
|------|--------|-----------------|
| Intra-VLAN connectivity | `ping` between hosts in same VLAN | Success |
| Inter-VLAN routing | `ping` across VLANs via SVI | Success |
| DHCP lease | `ipconfig /renew` on end hosts | IP assigned from correct pool |
| SSH login | `ssh -l admin <device-ip>` | Remote shell access |
| HQ ↔ Branch connectivity | `ping` across sites | Success via VPN tunnel |
| IPSec tunnel | `show crypto ipsec sa` | Packets encrypted/decrypted |
| PAT/Internet | Ping to ISP/public IP | NAT translation active |
| ACL enforcement | Traffic from restricted source | Denied by ACL rule |
| Wireless clients | Associate to AP, get DHCP | IP in correct VLAN range |
| Port security | Connect 2nd device to server switch | Port shuts down |

---

## 🛠️ Tools Used

- **Cisco Packet Tracer** — Network simulation and implementation
- Cisco routers (serial WAN links, OSPF, VPN, PAT)
- Cisco Catalyst multilayer switches (L3 routing + switching)
- Cisco Catalyst L2 access switches
- Cisco Access Points (wireless)
- Dedicated server devices (DHCP, DNS, Web, Email)

---

## 👤 Author

**Hisham** — Network Engineering & IT  
📍 Egypt | 🎓 CS Graduate, 6 October University  
🔗 [GitHub](https://github.com/thisishisham1)
