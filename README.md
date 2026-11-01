<div align="center">

# 🌐 EBGP (External BGP) Lab — Two Autonomous Systems Peering

### Border Gateway Protocol | AS 100 ⇄ AS 200 | GNS3 + Cisco IOS

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-BGP-orange?style=for-the-badge)
![Type](https://img.shields.io/badge/Peering-EBGP-red?style=for-the-badge)
![AD](https://img.shields.io/badge/Admin%20Distance-20-yellow?style=for-the-badge)
![Transport](https://img.shields.io/badge/TCP-179-blue?style=for-the-badge)
![Lab](https://img.shields.io/badge/Lab-GNS3-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA%20%E2%86%92%20CCNP-purple?style=for-the-badge)

*A hands-on lab that builds an External BGP session between two autonomous systems, advertises each LAN, and verifies the BGP table and the routing table.*

</div>

---

## 📑 Table of Contents

1. [Project Overview](#-project-overview)
2. [BGP Introduction](#-bgp-introduction)
3. [Types of BGP](#-types-of-bgp)
4. [BGP Administrative Distance](#-bgp-administrative-distance-ad)
5. [How BGP Behaves](#-how-bgp-behaves)
6. [Lab Topology](#-lab-topology)
7. [Addressing Table](#-addressing-table)
8. [Configuration](#-configuration)
9. [Verification (Stage by Stage)](#-verification-stage-by-stage)
10. [Command Cheat Sheet](#-command-cheat-sheet)
11. [Key Takeaways](#-key-takeaways)
12. [Troubleshooting Checklist](#-troubleshooting-checklist)
13. [Skills Demonstrated](#-skills-demonstrated)
14. [Author](#-author)

---

## 🎯 Project Overview

| Item | Details |
|------|---------|
| **Lab Name** | EBGP — External BGP between two Autonomous Systems |
| **Platform** | GNS3 (Cisco IOS routers, switches, VPCs), accessed over Telnet |
| **Autonomous Systems** | AS 100 (Router1 / R1) and AS 200 (Router2 / R2) |
| **Peering Link** | FastEthernet0/0 — `1.1.1.0/24` |
| **Advertised Networks** | `192.168.1.0/24` (AS 100) and `192.168.2.0/24` (AS 200) |
| **Goal** | Establish an EBGP neighbor relationship and exchange LAN prefixes between two different ASNs |
| **Verified With** | `%BGP-5-ADJCHANGE` log, `show ip bgp`, `show ip route` |

### ✅ What this lab proves

- Two routers in **different AS numbers** can become EBGP neighbors with just `neighbor <ip> remote-as <asn>`.
- The `network` command with the **exact prefix and mask** originates the LAN into BGP.
- Routes learned from an EBGP peer are installed in the routing table with **Administrative Distance 20**.

---

## 📘 BGP Introduction

**BGP (Border Gateway Protocol)** is the routing protocol that runs the Internet. It is the only **EGP (Exterior Gateway Protocol)** in use today and is defined in RFC 4271 (BGP-4).

| Property | Description |
|----------|-------------|
| **Category** | Exterior Gateway Protocol (routing *between* autonomous systems) |
| **Algorithm** | **Path Vector** (tracks the list of ASNs a route passed through) |
| **Transport** | **TCP port 179** — reliable, unicast, session-based |
| **Neighbors** | **Manually configured**, no auto-discovery like OSPF/EIGRP hello multicast |
| **Updates** | Incremental and triggered; no periodic full table refresh |
| **Metric** | No single metric — uses **path attributes** (Weight, Local Pref, AS-Path, Origin, MED, etc.) |
| **Loop Prevention** | A router rejects a route if its **own ASN** is already in the AS-Path |
| **Scalability** | Built for very large networks (the global Internet table) |
| **Focus** | **Policy and control**, not fastest convergence |

### 🆚 IGP vs EGP

| | IGP (OSPF, EIGRP, RIP) | EGP (BGP) |
|---|---|---|
| Scope | Inside **one** autonomous system | **Between** autonomous systems |
| Goal | Fast, optimal paths | Policy-based, scalable path selection |
| Neighbor discovery | Automatic (hello) | Manual `neighbor` statement |
| Transport | Own protocol / IP | TCP 179 |
| Metric | Cost / composite | Path attributes |

### 🔢 What is an Autonomous System (AS)?

A collection of networks under **one administrative control** that presents a **common routing policy** to the outside world. Each AS is identified by an **ASN**:

| Range | Type |
|-------|------|
| 1 – 64495 | Public 16-bit ASNs |
| 64512 – 65534 | **Private** 16-bit ASNs |
| 4-byte ASNs | Extended range (RFC 6793) |

> In this lab, **AS 100** and **AS 200** are used as lab AS numbers.

---

## 🔀 Types of BGP

| | **EBGP (External BGP)** | **IBGP (Internal BGP)** |
|---|---|---|
| **Neighbors are in** | **Different** AS numbers | **Same** AS number |
| **Config rule** | `remote-as` ≠ local AS | `remote-as` = local AS |
| **Administrative Distance** | **20** | **200** |
| **Default TTL** | **1** (neighbors normally directly connected) | 255 (can be multiple hops away) |
| **AS-Path** | Local ASN is **prepended** when advertising | AS-Path **not** changed |
| **Next-Hop** | Changed to the advertising router's own IP | **Not** changed by default |
| **Loop prevention** | AS-Path check | **Split-horizon rule** — routes learned from an IBGP peer are not passed to another IBGP peer (so a full mesh or route reflector is needed) |
| **Typical use** | ISP ⇄ customer, ISP ⇄ ISP | Distributing BGP info inside one AS |

> ⭐ **This lab is EBGP** — R1 (AS 100) peers with R2 (AS 200) across a directly connected link.

---

## 📏 BGP Administrative Distance (AD)

Administrative Distance decides **which routing source is trusted** when multiple protocols know the same prefix. Lower = more trusted.

| Route Source | AD |
|--------------|----|
| Connected | 0 |
| Static | 1 |
| **EBGP** | **20** ⭐ |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| **IBGP** | **200** |
| Unknown / unreachable | 255 |

### 🔎 Where AD 20 is visible in this lab

```
B     192.168.2.0/24 [20/0] via 1.1.1.2, 00:02:39
                      │  │
                      │  └── Metric (MED) = 0
                      └───── Administrative Distance = 20  → EBGP
```

> 💡 **Note:** The second number `0` is **not** a "BGP metric" that BGP uses to choose paths. It is the **MED/IGP-cost value** carried with the route (0 here because the prefix was originated from a directly connected network). BGP chooses paths using **attributes**, not one metric.

---

## ⚙️ How BGP Behaves

### 1️⃣ Neighbor (Session) Formation

BGP builds a **TCP session on port 179**, then exchanges messages:

| Message | Purpose |
|---------|---------|
| **OPEN** | Starts the session; carries ASN, hold time, router ID, capabilities |
| **KEEPALIVE** | Confirms the neighbor is alive (default every 60 s, hold time 180 s) |
| **UPDATE** | Advertises new prefixes and path attributes, or withdraws routes |
| **NOTIFICATION** | Reports an error and **closes** the session |

### 2️⃣ BGP Neighbor States (FSM)

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

| State | Meaning |
|-------|---------|
| Idle | Waiting to start a connection; no resources allocated |
| Connect | Waiting for TCP 3-way handshake to complete |
| Active | TCP attempt failed or retrying (often a reachability/config issue) |
| OpenSent | OPEN message sent, waiting for the peer's OPEN |
| OpenConfirm | OPEN received, waiting for KEEPALIVE |
| **Established** | ✅ Session up — UPDATEs are exchanged |

In this lab the log `%BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up` confirms the **Established** state.

### 3️⃣ Router ID

BGP needs a **Router ID**. If not set manually, IOS picks the **highest loopback IP**, otherwise the **highest active interface IP**.
In this lab R2's Router ID is **1.1.1.2** (seen in `show ip bgp`).

### 4️⃣ The `network` Command Behaves Differently in BGP

| In IGPs (OSPF/EIGRP) | In BGP |
|---|---|
| Enables the protocol on matching interfaces | **Does not enable anything on an interface** |
| Wildcard / classful matching | Advertises a prefix **only if an exact match exists in the routing table** |

So `network 192.168.2.0 mask 255.255.255.0` works because `192.168.2.0/24` is a **connected route** on R2.
If the prefix/mask doesn't exactly match a route in the RIB, **it will not be advertised**.

### 5️⃣ Path Selection — Important Attributes

BGP picks the best path using this order (simplified):

| Order | Attribute | Prefer |
|-------|-----------|--------|
| 1 | **Weight** (Cisco-only, local) | Highest |
| 2 | **Local Preference** | Highest |
| 3 | **Locally originated** | Network/aggregate on this router |
| 4 | **AS-Path** length | Shortest |
| 5 | **Origin** | IGP (`i`) < EGP (`e`) < Incomplete (`?`) |
| 6 | **MED** | Lowest |
| 7 | **EBGP over IBGP** | EBGP |
| 8 | **IGP metric to next hop** | Lowest |
| 9 | **Router ID** (tie-breaker) | Lowest |

### 6️⃣ Reading the BGP Table Columns

| Column | Meaning |
|--------|---------|
| `*` | Valid route |
| `>` | **Best** path (the one installed in the routing table) |
| `i` (status) | Learned via IBGP |
| Network | Destination prefix |
| Next Hop | IP to reach the prefix (`0.0.0.0` = locally originated) |
| Metric | MED |
| LocPrf | Local Preference (blank for EBGP-learned routes) |
| Weight | Cisco-local preference. **32768 = locally originated**, 0 = learned from a neighbor |
| Path | AS-Path (list of ASNs) |
| Origin code | `i` = IGP (via `network`), `e` = EGP, `?` = incomplete (redistribution) |

---

## 🗺️ Lab Topology

```
                      ┌──────────────── EBGP ────────────────┐
                      │                                       │
 ╔═══════════════ AS 100 ═══════════════╗      ╔═══════════════ AS 200 ═══════════════╗
 ║                                      ║      ║                                      ║
 ║            ┌──────────┐              ║      ║            ┌──────────┐              ║
 ║            │ Router1  │ Fa0/0 ───────╫──────╫─────── Fa0/0 │ Router2  │              ║
 ║            │  (R1)    │  1.1.1.1/24  ║      ║  1.1.1.2/24 │  (R2)    │              ║
 ║            └────┬─────┘              ║      ║            └────┬─────┘              ║
 ║              Fa1/0                   ║      ║              Fa1/0                   ║
 ║          192.168.1.1/24              ║      ║          192.168.2.1/24              ║
 ║                 │                    ║      ║                 │                    ║
 ║            ┌────┴─────┐              ║      ║            ┌────┴─────┐              ║
 ║            │ Switch3  │              ║      ║            │ Switch4  │              ║
 ║            └┬───┬───┬─┘              ║      ║            └┬───┬───┬─┘              ║
 ║         VPC5 VPC6 VPC7               ║      ║         VPC8 VPC9 VPC10              ║
 ╚══════════════════════════════════════╝      ╚══════════════════════════════════════╝
```

---

## 📋 Addressing Table

| Device | Interface | IP Address | Subnet Mask | Connected To |
|--------|-----------|------------|-------------|--------------|
| **R1** (AS 100) | Fa0/0 | 1.1.1.1 | 255.255.255.0 | R2 Fa0/0 |
| **R1** (AS 100) | Fa1/0 | 192.168.1.1 | 255.255.255.0 | Switch3 |
| **R2** (AS 200) | Fa0/0 | 1.1.1.2 | 255.255.255.0 | R1 Fa0/0 |
| **R2** (AS 200) | Fa1/0 | 192.168.2.1 | 255.255.255.0 | Switch4 |
| VPC5 – VPC7 | eth0 | 192.168.1.x | 255.255.255.0 | Switch3 |
| VPC8 – VPC10 | eth0 | 192.168.2.x | 255.255.255.0 | Switch4 |

### BGP Parameters

| Parameter | R1 | R2 |
|-----------|----|----|
| Local AS | 100 | 200 |
| Neighbor | 1.1.1.2 | 1.1.1.1 |
| Remote AS | 200 | 100 |
| Advertised Network | 192.168.1.0/24 | 192.168.2.0/24 |
| Peering Type | **EBGP** | **EBGP** |

---

## 🛠️ Configuration

### 🔹 Router1 (AS 100)

```bash
R1(config)# router bgp 100
R1(config-router)# network 192.168.1.0 mask 255.255.255.0
R1(config-router)# neighbor 1.1.1.2 remote-as 200
```

### 🔹 Router2 (AS 200)

```bash
R2(config)# router bgp 200
R2(config-router)# network 192.168.2.0 mask 255.255.255.0
R2(config-router)# neighbor 1.1.1.1 remote-as 100
```

### 📝 Command Explanation

| Command | What it does |
|---------|--------------|
| `router bgp <asn>` | Starts the BGP process with the local AS number |
| `network <prefix> mask <mask>` | Originates the prefix into BGP — **must match a route in the routing table exactly** |
| `neighbor <ip> remote-as <asn>` | Defines the peer. If `remote-as` ≠ local AS, the session is **EBGP** |

### ✅ Result right after configuring R2

```
*Nov  1 14:08:56.567: %BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up
```

The session came up immediately after `neighbor 1.1.1.1 remote-as 100` was entered on R2, because R1 had already been configured with the matching neighbor statement.

---

## 🔍 Verification (Stage by Stage)

### 🟢 Stage 1 — BGP Neighbor Came Up

**Command / Evidence:** console log on R2

```
R2(config-router)# neighbor 1.1.1.1 remote-as 100
*Nov  1 14:08:56.567: %BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up
```

| Annotation | Meaning |
|------------|---------|
| `%BGP-5-ADJCHANGE` | BGP adjacency state-change message (severity 5 = notification) |
| `neighbor 1.1.1.1 Up` | The TCP/179 session reached **Established** |

✅ **Result:** EBGP peering between AS 100 and AS 200 is established.

---

### 🟢 Stage 2 — BGP Table on R2 (`show ip bgp`)

```
R2# show ip bgp
BGP table version is 3, local router ID is 1.1.1.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  192.168.1.0      1.1.1.1                  0             0 100 i
 *>  192.168.2.0      0.0.0.0                  0         32768 i
```

#### 📌 Annotated Reading

| Marker | Where | Explanation |
|--------|-------|-------------|
| 🟥 `*>` | Both lines | `*` = **valid**, `>` = **best** path — installed in the routing table |
| 🟦 `i - IGP` / `i` | Origin code | Prefix was injected with the **`network` command** (origin = IGP) |
| 🟨 `100` | Path of `192.168.1.0` | **AS-Path = 100** — the route came from AS 100, one AS away |
| `1.1.1.1` | Next Hop | R1's interface IP — the EBGP next hop |
| `0.0.0.0` | Next Hop of `192.168.2.0` | Route is **locally originated** on R2 |
| `32768` | Weight | Default weight for **locally originated** routes (learned routes = 0) |
| `1.1.1.2` | Router ID | R2's BGP Router ID (highest active interface IP) |
| Empty LocPrf | Column | Local Preference is **not carried** across EBGP |
| Empty Path on `192.168.2.0` | Row 2 | Empty AS-Path = originated inside the local AS |

✅ **Result:** R2 learned `192.168.1.0/24` from AS 100 and also originates its own `192.168.2.0/24`.

---

### 🟢 Stage 3 — Routing Table on R1 (`show ip route`)

```
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       ...
Gateway of last resort is not set

      1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        1.1.1.0/24 is directly connected, FastEthernet0/0
L        1.1.1.1/32 is directly connected, FastEthernet0/0
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, FastEthernet1/0
L        192.168.1.1/32 is directly connected, FastEthernet1/0
B     192.168.2.0/24 [20/0] via 1.1.1.2, 00:02:39
```

#### 📌 Annotated Reading

| Marker | Explanation |
|--------|-------------|
| 🟥 `B - BGP` | Route code `B` = learned through **BGP** |
| 🟥 `B 192.168.2.0/24` | R1 learned **AS 200's LAN** from R2 |
| 🟨 `[20/0]` | **20 = Administrative Distance of EBGP**; 0 = MED/metric value carried with the route |
| `via 1.1.1.2` | Next hop = R2's Fa0/0 address |
| `00:02:39` | Age of the route (time since it was learned) |
| `C` / `L` | Connected and Local routes for the link and LAN |

✅ **Result:** The EBGP route is installed in R1's routing table with **AD 20**, so R1 can now forward traffic to the AS 200 LAN.

---

### 🔄 Stage Summary

| Stage | Check | Command | Result |
|-------|-------|---------|--------|
| 1 | Neighbor adjacency | Log `%BGP-5-ADJCHANGE` | ✅ Neighbor Up |
| 2 | BGP table | `show ip bgp` | ✅ `*>` best path, AS-Path 100, origin `i` |
| 3 | Routing table | `show ip route` | ✅ `B 192.168.2.0/24 [20/0]` |

---

## 🧾 Command Cheat Sheet

| Purpose | Command |
|---------|---------|
| Start BGP | `router bgp <asn>` |
| Advertise a network | `network <prefix> mask <mask>` |
| Create neighbor | `neighbor <ip> remote-as <asn>` |
| View BGP table | `show ip bgp` |
| View neighbor state | `show ip bgp summary` |
| View neighbor details | `show ip bgp neighbors` |
| View BGP routes in RIB | `show ip route bgp` |
| Check one prefix | `show ip bgp 192.168.2.0` |
| Reset a session | `clear ip bgp *` |
| Save config | `write memory` |

---

## 🧠 Key Takeaways

1. 🌍 **BGP is the Internet's routing protocol** — it connects autonomous systems and is a **path-vector** protocol.
2. 🔌 **BGP runs over TCP 179**; neighbors are **configured manually**, not discovered automatically.
3. 🔀 **EBGP = different ASNs, AD 20. IBGP = same ASN, AD 200.**
4. 🛣️ **AS-Path** shows which ASNs a route passed through and prevents loops (`100` appeared in R2's table).
5. 🎯 The BGP **`network` command advertises an exact prefix that already exists in the routing table** — it does not enable BGP on an interface.
6. ✅ In `show ip bgp`: **`*` = valid, `>` = best**, **`i` = origin IGP**, **Weight 32768 = locally originated**.
7. 📏 `[20/0]` in the routing table: **20 is the AD**, and `0` is the carried MED/metric value — BGP does **not** pick paths using a single metric.
8. 🧾 EBGP neighbors are normally **directly connected** (TTL = 1).
9. 🔔 The log `%BGP-5-ADJCHANGE ... Up` is the quickest confirmation that the session reached **Established**.
10. 🧱 Both sides must configure the **correct neighbor IP and remote AS**, or the session will not form.

---

## 🩺 Troubleshooting Checklist

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Neighbor stuck in **Idle / Active** | No IP reachability to neighbor | Check interface status and `ping` the neighbor |
| Neighbor never comes up | Wrong `remote-as` or wrong neighbor IP | Verify both sides match |
| Session flaps / drops | Layer-2 issue, ACL blocking TCP 179 | Check the link, ACLs, firewall |
| Prefix not in BGP table | `network` mask doesn't match the RIB exactly | Verify with `show ip route` and fix the mask |
| Route in BGP table but not in RIB | Better AD route exists, or next hop unreachable | Check `show ip bgp` and `show ip route` |
| Route missing after `clear` | Session still re-establishing | Wait and re-check `show ip bgp summary` |

---

## 💡 Ideas to Extend This Lab

- ➕ Configure **loopback-based peering** with `ebgp-multihop`
- 🔐 Add **MD5 authentication** with `neighbor <ip> password`
- 🧮 Add a **third AS** and observe longer AS-Paths
- 🎛️ Experiment with **Weight, Local Preference, MED, and AS-Path prepending**
- 🏠 Build **IBGP** inside one AS and compare AD 200 vs AD 20
- 🧪 Test end-to-end connectivity from VPC5 to VPC8 with `ping` and `trace`

---

## 🧰 Skills Demonstrated

`BGP` · `EBGP` · `Autonomous Systems` · `Path Vector Routing` · `AS-Path` · `Administrative Distance` · `Cisco IOS` · `GNS3` · `Routing Table Analysis` · `Network Troubleshooting` · `Network Documentation`

---

## 👤 Author

<div align="center">

**Maaz Khan**
CCNA-Certified Network & NOC Engineer
📍 Lower Dir, KPK, Pakistan

![CCNA](https://img.shields.io/badge/CCNA-Certified-blue?style=flat-square&logo=cisco)
![NOC](https://img.shields.io/badge/NOC-Engineer-orange?style=flat-square)

⭐ If this lab helped you, give the repo a star!

</div>

---

<div align="center">

**#BGP #EBGP #CCNA #CCNP #Cisco #GNS3 #Networking #NetworkEngineer #NOC**

</div>
