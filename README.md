<div align="center">

# 🌐 CSE421 Multi-Zone Enterprise Network Design

### VLSM-Based Network Architecture with Hybrid Routing, DHCP, DNS, Web & Email Services

<p>
  <img src="https://img.shields.io/badge/Cisco-Packet%20Tracer-blue?style=for-the-badge&logo=cisco&logoColor=white" />
  <img src="https://img.shields.io/badge/Routing-RIP%20v2%20%2B%20Static-success?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Subnetting-VLSM-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Services-DHCP%20%7C%20DNS%20%7C%20HTTP%20%7C%20SMTP%20%7C%20POP3-purple?style=for-the-badge" />
</p>

<p>
  A complete Cisco Packet Tracer project implementing a structured multi-zone network for Malcolm, Francis, Dewey, Reese, Hal, and the Family Home network.
</p>

</div>

---

## 📌 Project Overview

This project designs and implements a complete multi-zone network infrastructure in **Cisco Packet Tracer**.  
The network connects six major zones with efficient IP addressing, hybrid routing, DHCP automation, DNS name resolution, web hosting, and email communication.

The goal was not only to make devices ping each other, but to make real application-layer services work across multiple subnets.

> **Core idea:** A network is not truly working until routing, addressing, DNS, web, and email services all work together.

---

## 🧭 Network Zones

| Zone | Purpose | LAN Network | Gateway | Host Requirement |
|---|---|---:|---:|---:|
| Malcolm | School Lab | `10.109.0.0/22` | `10.109.0.1` | 840 |
| Francis | Military Academy | `10.109.4.0/23` | `10.109.4.1` | 390 |
| Dewey | Creative Experiment Lab | `10.109.6.0/23` | `10.109.6.1` | 300 |
| Reese | Kitchen Control Zone | `10.109.8.0/24` | `10.109.8.1` | 210 |
| Hal | Workplace Office | `10.109.9.0/25` | `10.109.9.1` | 72 |
| Home | Family Home Network | `10.109.9.128/27` | `10.109.9.129` | 25 |

---

## 🖼️ Topology Summary

```text
                 ┌──────────────┐
                 │  R-Malcolm   │
                 │ 10.109.0.1   │
                 └──────┬───────┘
                        │
          ┌─────────────┴─────────────┐
          │                           │
      ┌───▼───┐                   ┌───▼───┐
      │ R-Hal │                   │R-Reese│
      └───┬───┘                   └───┬───┘
          │                           │
       SW-Hal                     R-Francis
          │                           │
     Hal PCs + DHCP              SW-Francis
                                      │
                                  R-Home
                                  /    \
                              Home LAN  R-Dewey
                                │         │
                 DNS + Web + PCs     Dewey PCs + Email + DHCP
```

Each zone contains:
- A router as the default gateway
- A switch for local LAN connectivity
- At least two end devices
- Required services based on the assignment specification

---

## 🧮 VLSM Addressing Plan

Base network was derived from the student ID:

```text
Student ID: 23101109
Last three digits: 109
Base Network: 10.109.0.0/16
```

### VLSM Allocation

| Network | Hosts Needed | Prefix | Subnet Mask | Total Addresses | Usable Hosts |
|---|---:|---:|---|---:|---:|
| Malcolm | 840 | `/22` | `255.255.252.0` | 1024 | 1022 |
| Francis | 390 | `/23` | `255.255.254.0` | 512 | 510 |
| Dewey | 300 | `/23` | `255.255.254.0` | 512 | 510 |
| Reese | 210 | `/24` | `255.255.255.0` | 256 | 254 |
| Hal | 72 | `/25` | `255.255.255.128` | 128 | 126 |
| Home | 25 | `/27` | `255.255.255.224` | 32 | 30 |
| Router Links | 2 each | `/30` | `255.255.255.252` | 4 | 2 |

### Why VLSM?

VLSM was used because every zone required a different number of hosts.  
Allocating the same subnet size to all zones would waste a large number of IP addresses.

Example:

```text
Malcolm needs 840 hosts.
2^10 - 2 = 1022 usable hosts.
So Malcolm requires 10 host bits.
32 - 10 = /22.
```

---

## 🔗 Router-to-Router Links

All router-to-router links use `/30` subnets because each point-to-point link only needs two usable IP addresses.

| Link | Network | Router A | Router B |
|---|---|---:|---:|
| Malcolm ↔ Hal | `10.109.9.160/30` | `10.109.9.161` | `10.109.9.162` |
| Francis ↔ Reese | `10.109.9.164/30` | `10.109.9.165` | `10.109.9.166` |
| Reese ↔ Malcolm | `10.109.9.168/30` | `10.109.9.169` | `10.109.9.170` |
| Home ↔ Francis | `10.109.9.172/30` | `10.109.9.173` | `10.109.9.174` |
| Home ↔ Dewey | `10.109.9.180/30` | `10.109.9.181` | `10.109.9.182` |

> Router-to-router Ethernet connections were made using **copper cross-over cables** because the links are between same-type devices using GigabitEthernet interfaces.

---

## 🧠 Routing Design

This project uses a **hybrid routing architecture**:

- **RIP v2** for dynamic routing
- **Static routes** for controlled paths
- **Default route** for unknown traffic
- **Floating static route** for backup/failover logic

### RIP v2 Configuration

```cisco
router rip
 version 2
 network 10.0.0.0
 no auto-summary
```

### Why `no auto-summary`?

The project uses VLSM subnets like `/22`, `/23`, `/25`, and `/30`.  
Without `no auto-summary`, RIP may summarize routes incorrectly and break connectivity.

### Static Route Example

```cisco
ip route 10.109.6.0 255.255.254.0 10.109.9.165
```

### Default Route Example

```cisco
ip route 0.0.0.0 0.0.0.0 10.109.9.174
```

### Floating Static Route Example

```cisco
ip route 10.109.6.0 255.255.254.0 10.109.9.182 25
```

The administrative distance of `25` makes it a backup route.

---

## 🧾 DHCP Design

DHCP was implemented in two ways.

### Router-Based DHCP

| Zone | DHCP Source |
|---|---|
| Malcolm | R-Malcolm |
| Reese | R-Reese |

Example:

```cisco
ip dhcp excluded-address 10.109.0.1 10.109.0.10

ip dhcp pool MALCOLM
 network 10.109.0.0 255.255.252.0
 default-router 10.109.0.1
 dns-server 10.109.9.130
```

### Server-Based DHCP

| Zone | DHCP Server |
|---|---|
| Francis | DHCPSrv-Francis |
| Hal | DHCPSrv-Hal |
| Dewey | DHCPSrv-Dewey |

Each DHCP pool provides:

```text
IP Address
Subnet Mask
Default Gateway
DNS Server
```

### DHCP Flow

```text
Discover → Offer → Request → Acknowledge
```

---

## 🌍 DNS Configuration

The DNS server is hosted in the Home network.

| DNS Server | IP |
|---|---:|
| DNSSrv-Home | `10.109.9.130` |

### DNS Records

| Domain | Address |
|---|---:|
| `malcolm.com` | `10.109.0.2` |
| `mail.malcolm.com` | `10.109.0.2` |
| `dewey.com` | `10.109.6.3` |
| `mail.dewey.com` | `10.109.6.3` |
| `wilkersonhub.home.local` | `10.109.9.131` |

DNS is required because email and web services rely on domain-name resolution.

---

## 🌐 Web Server Configuration

The Home network contains a dedicated web server named **WilkersonHub**.

| Server | IP | Gateway | DNS |
|---|---:|---:|---:|
| WilkersonHub | `10.109.9.131` | `10.109.9.129` | `10.109.9.130` |

### Web Page Message

```text
Life is unfair... but the network works!
```

### Test

```text
http://10.109.9.131
http://wilkersonhub.home.local
```

---

## 📧 Email Service Configuration

Email communication was configured between Malcolm and Dewey.

### Malcolm Email Server

| Field | Value |
|---|---|
| IP | `10.109.0.2` |
| Gateway | `10.109.0.1` |
| DNS | `10.109.9.130` |
| Domain | `malcolm.com` |
| Mail Host | `mail.malcolm.com` |

### Dewey Email Server

| Field | Value |
|---|---|
| IP | `10.109.6.3` |
| Gateway | `10.109.6.1` |
| DNS | `10.109.9.130` |
| Domain | `dewey.com` |
| Mail Host | `mail.dewey.com` |

### Email Protocols

| Protocol | Purpose |
|---|---|
| SMTP | Sends email |
| POP3 | Receives email |

### Email Flow

```text
PC-Malcolm
   ↓ SMTP
EmailSrv-Malcolm
   ↓ SMTP
EmailSrv-Dewey
   ↓ POP3
PC-Dewey
```

---

## 🧪 Verification Commands

### PC Commands

```bash
ipconfig
ipconfig /all
ping 10.109.0.1
ping 10.109.6.3
ping dewey.com
nslookup mail.dewey.com
tracert 10.109.6.3
```

### Router Commands

```cisco
show ip interface brief
show ip route
show ip route rip
show ip protocols
show running-config
```

### Expected Traceroute Path

```text
Malcolm → Reese → Francis → Home → Dewey
```

---

## 🛠️ Major Issues Faced & Fixes

### DNS Failure

**Problem**

```text
ping 10.109.6.3 worked
ping dewey.com failed
```

**Cause**

DNS records were missing or clients had the wrong DNS server.

**Fix**

```text
dewey.com → 10.109.6.3
mail.dewey.com → 10.109.6.3
DNS Server → 10.109.9.130
```

### DHCP Pool Conflict

**Problem**

Clients received wrong DNS `10.109.9.129`.

**Cause**

Multiple DHCP pools existed for the same subnet.

**Fix**

Removed duplicate/wrong pool and kept the correct pool with:

```cisco
dns-server 10.109.9.130
```

### Email Delivery Failure

**Problem**

Email generated DNS error.

**Cause**

Mail domain could not resolve to the destination email server.

**Fix**

Configured:

```text
mail.malcolm.com → 10.109.0.2
mail.dewey.com → 10.109.6.3
```

### PC Interface Conflict

**Problem**

PC-Reese1 received IP on Bluetooth instead of FastEthernet.

**Cause**

Packet Tracer interface conflict.

**Fix**

Disabled/removed the unused interface or recreated the PC and assigned DHCP to FastEthernet.

---

## 📚 Concepts Demonstrated

- VLSM subnetting
- Router-to-router addressing using `/30`
- RIP v2 dynamic routing
- Static routing
- Default routing
- Floating static routing
- Router-based DHCP
- Server-based DHCP
- DNS resolution
- HTTP service hosting
- SMTP and POP3 email communication
- Ping and traceroute troubleshooting

---

## 🧠 Viva Notes

### Why VLSM?

Because each network required a different number of hosts, so VLSM reduces address waste.

### Why RIP v2?

Because RIP v2 supports classless routing and works with VLSM.

### Why static routes?

Static routes give manual control over selected traffic paths.

### Why default route?

It forwards unknown traffic to a chosen next-hop router.

### Why floating static route?

It provides backup connectivity when the primary route fails.

### Why DNS?

DNS translates human-readable domain names into IP addresses.

### Why DHCP?

DHCP automatically provides IP, subnet mask, gateway, and DNS to clients.

### Why cross-over cable?

Router-to-router Ethernet connections are same-device connections, so cross-over cables align transmit and receive pins.

---

## ✅ Final Result

The final network supports:

- Full inter-zone connectivity
- Working DHCP across required zones
- Static IP addressing in Home
- DNS-based name resolution
- Web server access
- Email communication using SMTP and POP3
- Verified routing using ping and traceroute

---

## 🛠️ Tools Used

<p>
  <img src="https://img.shields.io/badge/Cisco%20Packet%20Tracer-Network%20Simulation-blue?style=flat-square" />
  <img src="https://img.shields.io/badge/Cisco%20IOS-CLI-informational?style=flat-square" />
  <img src="https://img.shields.io/badge/Networking-VLSM%20%7C%20RIP%20v2%20%7C%20DHCP%20%7C%20DNS-success?style=flat-square" />
</p>

---

## 👥 Contributors

| Member | Contribution |
|---|---|
| Maisha | VLSM planning, subnet allocation, addressing table |
| Shafin | Routing design, router configuration, troubleshooting |
| Fardin Saurov | DHCP planning, device addressing, client configuration |
| Jahid | DNS, web server, email service configuration |

---

## 📎 Suggested Repository Structure

```text
CSE421-Multi-Zone-Network/
│
├── README.md
├── topology/
│   └── network-topology.png
├── packet-tracer/
│   └── CSE421_Project_Final.pkt
├── documentation/
│   └── final-report.pdf
├── configs/
│   ├── R-Malcolm.txt
│   ├── R-Reese.txt
│   ├── R-Francis.txt
│   ├── R-Home.txt
│   ├── R-Dewey.txt
│   └── R-Hal.txt
└── screenshots/
    ├── web-server-working.png
    ├── email-test.png
    ├── dns-test.png
    └── routing-table.png
```

---

<div align="center">

## ⭐ Key Takeaway

**A successful ping does not always mean the whole network works.**  
Real connectivity means routing, DNS, DHCP, web, and email all working together.

</div>
