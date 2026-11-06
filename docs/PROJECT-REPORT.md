# Project Report: Billy's Bar & Grill Network

**Author:** Gokan "Shark" Bektas
**Role in the scenario:** Network consultant (my own IT company)
**Tools:** Cisco Packet Tracer 9.0.1
**Build dates:** November 5–6, 2026 (two sessions, about 7 hours of hands-on work)

---

## Contents

1. [Summary](#1-summary)
2. [The client and what they asked for](#2-the-client-and-what-they-asked-for)
3. [Constraints: budget and the owner's preferences](#3-constraints-budget-and-the-owners-preferences)
4. [Design decisions and why I made them](#4-design-decisions-and-why-i-made-them)
5. [Lab gear vs what I'd actually buy](#5-lab-gear-vs-what-id-actually-buy)
6. [How I built it, step by step](#6-how-i-built-it-step-by-step)
7. [Troubleshooting: what broke and how I fixed it](#7-troubleshooting-what-broke-and-how-i-fixed-it)
8. [Testing and results](#8-testing-and-results)
9. [Limitations and what I'd do next](#9-limitations-and-what-id-do-next)
10. [Skills this project covers](#10-skills-this-project-covers)
11. [What I learned](#11-what-i-learned)

---

## 1. Summary

I designed and built a complete network for a restaurant with a bar: a scenario I treated like a real client job. The finished network has 9 VLANs, inter-VLAN routing on a Layer 3 core switch, central DHCP and DNS, an ASA firewall with NAT and a DMZ hosting the restaurant's public website, a wireless controller for staff Wi-Fi, an isolated guest Wi-Fi, security cameras, and a full security lockdown (SSH-only management, port security, ACLs, and service hardening).

The most valuable part was the troubleshooting. I ran into six real problems, including guest Wi-Fi users landing in the management network and a firewall feature Packet Tracer doesn't support. I tracked each one down with show commands and Simulation mode, fixed it, and documented it.

**Final result:** every planned feature works. Guests get internet only, POS terminals are isolated like the PCI-DSS card standard expects, cameras can't reach the internet, and the website is reachable from outside while the DMZ can't touch the inside network.

---

## 2. The client and what they asked for

> This is a practice scenario. Billy's Bar & Grill is a fictional client. I wrote the brief at the start of the project and used it to drive every decision.

**The business:** one location with a dining room (front of house), a bar, a kitchen, and a back office. Card payments at the bar and at tables, kitchen and bar ticket printers, and a manager who handles the books from the office.

**What the owner asked for:**

| # | Requirement |
|---|---|
| R1 | Free Wi-Fi for customers |
| R2 | Separate networks for the bar and the restaurant side |
| R3 | Staff Wi-Fi for order-taking tablets |
| R4 | Security cameras, viewable from the office |
| R5 | Shared files (menus, schedules, invoices) |
| R6 | Mail server and a public website (online menu) |
| R7 | A firewall protecting the business |

**What I added as the consultant (the "better ideas"):**

| # | Addition | Why |
|---|---|---|
| A1 | Separate POS network | Card data should be isolated (PCI-DSS) |
| A2 | Cameras on their own network with no internet | IP cameras are a common botnet target |
| A3 | DMZ for public servers | If the website gets hacked, the attacker can't reach the business |
| A4 | Central logging and time sync | You can't investigate an incident without logs and matching timestamps |
| A5 | Secure management (SSH, passwords, banners) | Default switches have no passwords at all |

---

## 3. Constraints: budget and the owner's preferences

A restaurant isn't a bank. Margins are thin, there's no IT staff, and the owner wants things to just work. These constraints shaped the design as much as the technical requirements did.

| Constraint | What it meant for the design |
|---|---|
| **Small budget** | Buy only what's needed. One firewall, one core switch, and no redundant hardware. Spend money where risk is highest (firewall, POS isolation) and save it where risk is low. |
| **Non-technical owner** | A clean, predictable design: VLAN numbers that mean something, one DHCP server, one place for logs, and descriptions on every port so the next tech can follow it. |
| **Card payments** | POS isolation isn't optional. A breach costs far more than the network. |
| **Guests on the Wi-Fi every night** | Guests must never share a network with anything the business runs. |
| **Busy service hours** | Changes have to be safe. Port security uses "restrict" (drop and log) instead of "shutdown," so one mistake doesn't kill a POS terminal mid-dinner rush. |
| **Room to grow** | The /23 address plan has room, and the design can add a second location later (OSPF + VPN). |
| **Only one internet connection** | All staff share one public IP through PAT. The ISP's small static block (/29) covers the public servers. |

---

## 4. Design decisions and why I made them

### 4.1 Topology: collapsed core

```
Internet → ISP → Firewall → CORE (Layer 3) → 3 access switches (dining, bar, back of house)
                     └→ DMZ (web + mail)
```

**Why this shape:** a restaurant this size doesn't need a separate core and distribution layer. A **collapsed core** (one Layer 3 switch doing both jobs) is the standard small-business design. It's cheaper, has fewer devices to manage, and still gives clean separation.

**Why one access switch per area:** the switches match the physical building (dining room, bar, kitchen/office). Each closet serves the devices near it, so cable runs stay short.

### 4.2 VLANs by function, not by department

The client said "separate bar and restaurant," but a bar POS and a restaurant POS need **the same** protection. So I segmented by **what the device does**:

| VLAN | Purpose | Why it's separate |
|---|---|---|
| 20 POS | Card terminals + ticket printers | Card data isolation |
| 30 STAFF | Manager PC, host stand, office printer | Business work |
| 40 SERVERS | DHCP/DNS, files, logs | Protect the services everything depends on |
| 50 CCTV | Cameras + NVR | Untrusted IoT devices |
| 60 STAFF-WIFI | Staff tablets | Wireless is less trusted than wired |
| 70 GUEST-WIFI | Customers | Completely untrusted |
| 10 MGMT | Switches, WLC, APs | Only admins belong here |
| 99 NATIVE | Unused native VLAN | Stops VLAN hopping attacks |
| 999 BLACKHOLE | All unused ports, shut down | A random wall jack can't get on the network |

The bar-vs-restaurant split still exists **physically** (separate switches), and security policy follows function. Fewer VLANs means fewer rules, and fewer rules means fewer mistakes.

### 4.3 Inter-VLAN routing on the core switch

I chose a **Layer 3 switch with SVIs** instead of router-on-a-stick, because:
- it routes in hardware, so it's faster
- there's no single router link carrying every VLAN
- one fewer device to buy

### 4.4 IP addressing: VLSM from one /23

The whole site fits in **10.10.0.0/23**, sized with VLSM so nothing is wasted. The largest subnet came first (guest /24), and the smallest ones got /29 and /30. Every gateway is the **first usable address**, so anyone troubleshooting can guess it.

### 4.5 One central DHCP + DNS server

Every VLAN gets addresses from one server (SRV-CORE) through `ip helper-address` on the core. One place to manage instead of 6, and the owner's tech only has to look in one spot. Printers and the NVR use **static** IPs so nothing loses track of them.

### 4.6 Firewall with three zones

| Zone | Security level | Who lives there |
|---|---|---|
| outside | 0 | The internet |
| dmz | 50 | Public website + mail |
| inside | 100 | The whole restaurant |

The ASA allows high → low traffic by default and blocks low → high unless a rule allows it. Staff can browse, the internet can only reach the two published ports (80 and 25), and the DMZ can't start connections to the inside.

### 4.7 One public IP for staff, a small block for servers

- **Staff browsing:** PAT, so everyone shares the firewall's public IP. Free.
- **Public servers:** a **/29 static block** from the ISP with 1-to-1 static NAT (web = .3, mail = .4). I originally planned port forwarding on one IP, but Packet Tracer's ASA doesn't support it (see troubleshooting #3). In a real deal this is a small monthly add-on from the ISP.

### 4.8 Wireless

- **Staff Wi-Fi:** WLC-2504 + lightweight APs, so it's configured once and pushed to every AP. I also wanted CCNA practice with a controller.
- **Guest Wi-Fi:** a **dedicated AP plugged straight into VLAN 70**. Physically, a guest can't land anywhere except the guest network, and the GUEST-IN ACL limits them to internet only. Lots of small venues run guest Wi-Fi exactly like this.
- **WPA2-PSK with a printed password** for guests. A splash page is nicer, but it's more to maintain and harder in Packet Tracer.

### 4.9 Security layers (defense in depth)

| Layer | What I did |
|---|---|
| Edge | Firewall zones, only ports 80 and 25 open, stateful inspection |
| Network | ACLs on every user VLAN (POS, CCTV, STAFF, STAFF-WIFI, GUEST) |
| Switch port | Port security (sticky MAC, max 1), unused ports shut down in VLAN 999 |
| Device access | `enable secret`, local user, SSH v2 only, timeouts, login banner, encrypted passwords |
| Servers | Unneeded services turned off, default FTP account deleted, least-privilege FTP users |
| Visibility | All switches send Syslog to one server, all clocks synced with NTP |

---

## 5. Lab gear vs what I'd actually buy

Packet Tracer only has certain models, and several of them are old or end-of-life. For the real client, I'd swap in current equivalents and make a few budget-driven changes:

| In the lab | For the real client | Why |
|---|---|---|
| ASA 5506-X | **Cisco Secure Firewall 1010** (Cisco sells a small-business bundle with a 3-year license) | Current small-business firewall; the 5506-X is end-of-life |
| 3560-24PS core | A current **Layer 3–capable managed switch** | Same job, supported model |
| 2960 access switches | **24-port PoE+** access switches | PoE powers the APs and cameras over the network cable. In the lab I had to add power adapters to the APs because the 2960 doesn't do PoE |
| WLC-2504 + 3702i APs | **Wi-Fi 6 APs with the controller built in** (e.g., Cisco's Embedded Wireless Controller on Catalyst 9100 APs) | For 3 APs, a separate controller box isn't worth the money. I used the WLC in the lab for CCNA practice |
| 3 separate servers | **One small server or NAS** running DHCP/DNS, files, and logs | Cheaper, less power, one box to back up. They're separate in the lab only to make each role easy to see |
| On-site mail server | **Hosted email** (Microsoft 365 or Google Workspace) | Running your own mail server is a security and spam headache for a restaurant. I kept it in the lab for the DMZ practice |
| Webcams + NVR | PoE IP cameras + NVR | Same design, real hardware |

**Where I'd spend money:** the firewall (it protects everything) and PoE switches (no power bricks, and cleaner installs).
**Where I'd save:** no redundant core, no hardware controller, one combined server, hosted email.

Network gear pricing is quote-based through Cisco partners, so the next real step would be a formal quote. These choices keep the total as low as possible without cutting security corners.

---

## 6. How I built it, step by step

I built it in phases and tested each one before moving on, and saved a `.pkt` snapshot at each checkpoint (see [`pkt/`](../pkt)).

| Step | What I did | How I proved it worked |
|---|---|---|
| **1. Physical** | Placed ~35 devices, renamed every one, cabled with the right cable types (cross-over for like devices, straight-through for unlike) | All links green except the firewall (not configured yet) |
| **2. VLANs + trunks** | Created 9 VLANs on 4 switches, assigned access ports, built 802.1Q trunks with native VLAN 99 and an allowed-VLAN list | `show vlan brief`, `show interfaces trunk` showing all VLANs forwarding |
| **3. Routing** | `ip routing` + 7 SVIs on CORE, static IPs on servers | Server pinged the POS gateway in another VLAN |
| **4. DHCP** | 6 pools on SRV-CORE, `ip helper-address` on every user SVI, static printers/NVR | POS-R1 got 10.10.1.40 automatically; TTL 127 vs 128 proved routed vs same-VLAN traffic |
| **5. Services** | DNS A records, FTP with 2 users (default account deleted), switch management IPs, Syslog, NTP | `ping files.billysbar.local` resolved; FTP login; logs from all 4 switches with the correct date |
| **6. Wireless** | WLC setup, staff/guest interfaces and WLANs, APs joined via DHCP option 43 | Both APs registered (REG) on the WLC |
| **7. Firewall** | ISP router, ASA zones, routes, PAT, inspection, DMZ with static NAT, outside ACL | `show xlate` showed NAT working; MANAGER reached the internet (TTL 125); the website loaded **from the internet** at 203.0.113.3 |
| **8. Lockdown** | SSH, passwords, banners, blackhole VLAN, port security, 5 ACLs, firewall password, service hardening | Every ACL tested both ways (allowed and blocked) |
| **9. Guest fix** | Replaced the guest AP with an 802.11n model | Guest phone got 10.10.0.13; internet works, inside blocked |

---

## 7. Troubleshooting: what broke and how I fixed it

Full details: [`troubleshooting-log.md`](troubleshooting-log.md). The short version:

| # | Problem | How I found it | Fix |
|---|---|---|---|
| 1 | **Guest Wi-Fi users got management-VLAN addresses** (10.10.1.x instead of 10.10.0.x), a real security risk | Ruled out the WLAN interface, AP group mapping, DHCP proxy, trunking, and AP reboots one by one. A standalone guest AP showed an **empty MAC table** on its port, proving nothing was passing through it | Replaced the old **AP-PT** with an **AP-PT-N** (802.11n, matching the clients). Guests got 10.10.0.x right away |
| 2 | **Website not reachable from the internet** ("Server Reset Connection") | The `^` error pointed at `service`: Packet Tracer's ASA doesn't support port forwarding | Moved to a /29 public block with 1-to-1 static NAT, which is how a real business with public servers does it anyway |
| 3 | **Web browsing timed out while ping worked** | Simulation mode trace (HTTP/TCP filter) showed the first attempts dying mid-path, then success | Root cause was ARP warm-up across 5 hops, not a broken config. Removed the test rule I'd added, since it wasn't needed |
| 4 | **Firewall rule pointed at the wrong server** (.138 instead of .139) | Reviewing `show access-list` after the change | Removed the wrong rule, added the right one. Hit counters confirmed the correct rules were in use |
| 5 | **Firewall commands silently rejected** | The prompt was `FW>` instead of `FW#` | Habit: always check the prompt before pasting |
| 6 | **Cameras didn't answer pings from other VLANs** | Same-VLAN NVR could ping them, so the cameras were fine | Packet Tracer IoT limitation. The only traffic my design allows (camera ↔ NVR) works |

**My troubleshooting method:** start at the device and work outward (own IP → gateway → other VLAN → internet), change one thing at a time, and test after every change. When guessing stopped working, I used Simulation mode to watch where packets actually died.

---

## 8. Testing and results

Every security rule was tested in **both directions**: what should work, and what should be blocked. The full matrix with screenshots is in [`test-plan.md`](test-plan.md).

| Test | Expected | Result |
|---|---|---|
| Staff PC → internet | Works | ✅ TTL 125 |
| Internet → Billy's website (203.0.113.3) | Works | ✅ Page loads |
| DMZ web server → inside network | Blocked | ✅ |
| POS → internet / → office | Works / Blocked | ✅ / ✅ |
| Cameras' NVR → internet | Blocked | ✅ |
| Manager → NVR footage | Works | ✅ |
| Staff → POS | Blocked | ✅ |
| Guest phone → internet / → office / → POS | Works / Blocked / Blocked | ✅ / ✅ / ✅ |
| SSH to switches from MANAGER | Works with login + banner | ✅ |

---

## 9. Limitations and what I'd do next

**Known limitations**
- FTP sends passwords in clear text (Packet Tracer has no SFTP). Real deployment → SFTP or a NAS share over SMB.
- The website is HTTP only. Next step → HTTPS on 443 and close port 80.
- DMZ-MAIL has its IP, NAT, and firewall rule, but the mail service isn't configured. For the real client I'd recommend hosted email anyway.
- Guest Wi-Fi runs on a standalone AP, not through the WLC (Packet Tracer behavior, see troubleshooting #1).
- Lab passwords are written in the configs on purpose so others can open the lab. In a real network they'd never be published.

**Next improvements**
- WPA2-Enterprise with RADIUS for staff Wi-Fi (individual logins instead of a shared password)
- DHCP snooping + Dynamic ARP Inspection on the access switches
- Voice VLAN with IP phones at the host stand
- A second location with OSPF and a site-to-site VPN
- Config backups to a TFTP/SCP server

---

## 10. Skills this project covers

Mapped to the CCNA (200-301) exam areas:

| CCNA area | Where it shows up here |
|---|---|
| Network fundamentals | Cabling types, IPv4 addressing, VLSM, subnetting, TTL/hop reasoning |
| Network access | VLANs, 802.1Q trunks, native VLAN, STP behavior, WLC + lightweight APs, WPA2 |
| IP connectivity | Inter-VLAN routing with SVIs, routed ports, static and default routes |
| IP services | DHCP + relay, DNS, NAT/PAT, static NAT, NTP, Syslog |
| Security fundamentals | Extended ACLs, port security, SSH, password hardening, device access control, firewall zones, DMZ |
| Troubleshooting | Show commands, Simulation mode packet tracing, isolating problems layer by layer |

---

## 11. What I learned

- **Design for the business, not for the lab.** The client said "bar and restaurant," but the real requirement was protecting card data. Grouping by function made the security simpler.
- **Test the "blocked" side too.** A rule that's supposed to block something isn't proven until you try to get through it and fail.
- **Read the error.** The `^` under `service` told me exactly what was wrong. Twenty minutes of guessing could have been two minutes of reading.
- **Review before you trust.** One wrong digit in a firewall rule opened a port on the wrong server.
- **Know when it's the tool, not you.** Both the camera pings and the guest AP turned out to be Packet Tracer device quirks. But I only knew that after ruling out my own config step by step.
- **Document as you go.** Saving a snapshot after every step is why this repo can show the whole build.
