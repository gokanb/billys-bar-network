# Test plan and results

Every feature was tested after it was built. Security rules were tested **both ways**: the traffic that should pass, and the traffic that should be blocked.

**Legend:** ✅ passed · ⚠️ passed with a documented Packet Tracer limitation

## Connectivity

| # | Test | From → To | Expected | Result | Evidence |
|---|---|---|---|---|---|
| C1 | Trunks forwarding all VLANs | CORE `show interfaces trunk` | Fa0/1–3 forwarding 10–99 | ✅ | [04](../screenshots/04-core-trunks-forwarding.png) |
| C2 | All gateways up | CORE `show ip interface brief` | Vlan10–70 up/up | ✅ | [05](../screenshots/05-core-svis-up.png) |
| C3 | Inter-VLAN routing | SRV-CORE → POS gateway 10.10.1.33 | Reply | ✅ | [06](../screenshots/06-intervlan-routing-ping.png) |
| C4 | DHCP across VLANs (relay) | POS-R1 requests IP | 10.10.1.40 from POS pool | ✅ | [08](../screenshots/08-pos-dhcp-success.png) |
| C5 | Routed vs same-VLAN | MANAGER → printers/NVR | TTL 127 routed, 128 local | ✅ | [09](../screenshots/09-cross-vlan-ping-ttl.png) |
| C6 | DNS | MANAGER `ping files.billysbar.local` | Resolves to 10.10.1.131 | ✅ | [10](../screenshots/10-dns-resolution.png) |
| C7 | File share | MANAGER `ftp files.billysbar.local` as manager | Login + `dir` | ✅ | [11](../screenshots/11-ftp-login.png) |
| C8 | Central logging | All switches → SRV-MGMT | Logs from 4 switches, correct date (NTP) | ✅ | [12](../screenshots/12-syslog-all-switches.png) |
| C9 | APs join controller | WLC → All APs | 2 APs, status REG | ✅ | [14](../screenshots/14-wlc-aps-joined.png) |
| C10 | Firewall up | FW → ISP 203.0.113.1 | Reply | ✅ | [17](../screenshots/17-asa-interfaces-ping-isp.png) |
| C11 | PAT translation | FW `show xlate` during ping | 10.10.1.70 → 203.0.113.2 | ✅ | [18](../screenshots/18-asa-nat-xlate.png) |
| C12 | Staff internet | MANAGER → 198.51.100.10 | Reply, TTL 125 | ✅ | [19](../screenshots/19-internet-ping-ttl125.png) |
| C13 | Staff web browsing | MANAGER browser → http://198.51.100.10 | Page loads | ✅ | [21](../screenshots/21-internet-web-browsing.png) |
| C14 | Public website | WEB-INTERNET browser → http://203.0.113.3 | Billy's homepage | ✅ | [25](../screenshots/25-website-from-internet.png) |
| C15 | Camera ↔ NVR | NVR → CAM-FOH1 | Reply | ⚠️ Works same-VLAN; PT webcams ignore cross-VLAN ICMP | — |
| C16 | Guest Wi-Fi addressing | Guest phone DHCP | 10.10.0.x /24, gw 10.10.0.1 | ✅ | [36](../screenshots/36-guest-wifi-fixed-ip.png) |

## Security

| # | Test | From → To | Expected | Result | Evidence |
|---|---|---|---|---|---|
| S1 | DMZ can't reach inside | DMZ-WEB → MANAGER | Blocked | ✅ | [26](../screenshots/26-dmz-isolation-test.png) |
| S2 | Only needed ports open | FW `show access-list` | echo-reply, www→.138, smtp→.139 | ✅ | [27](../screenshots/27-asa-acl-hitcounts.png) |
| S3 | POS → internet | POS-R1 → 198.51.100.10 | Allowed | ✅ | [31](../screenshots/31-acl-pos-isolation.png) |
| S4 | POS → office | POS-R1 → MANAGER | Blocked | ✅ | [31](../screenshots/31-acl-pos-isolation.png) |
| S5 | Manager → camera footage | MANAGER → NVR | Allowed | ✅ | [32](../screenshots/32-acl-cctv-no-internet.png) |
| S6 | Cameras → internet | NVR → 198.51.100.10 | Blocked | ✅ | [32](../screenshots/32-acl-cctv-no-internet.png) |
| S7 | Staff → POS | MANAGER → KITCHEN-PRN | Blocked | ✅ | [33](../screenshots/33-acl-staff-test.png) |
| S8 | Staff → internet | MANAGER → 198.51.100.10 | Allowed | ✅ | [33](../screenshots/33-acl-staff-test.png) |
| S9 | Admin SSH | MANAGER → SW-FOH | Login prompt (only MANAGER may manage) | ✅ | [33](../screenshots/33-acl-staff-test.png) |
| S10 | SSH + banner | MANAGER `ssh -l admin 10.10.1.113` | Banner, then `CORE#` | ✅ | [28](../screenshots/28-ssh-login-banner.png) |
| S11 | Unused ports disabled | SW-FOH `show vlan brief` | Unused ports in VLAN 999, VLAN 1 empty | ✅ | [29](../screenshots/29-blackhole-vlan-999.png) |
| S12 | Port security on | SW-BOH `show port-security` | Max 1, restrict | ✅ | [30](../screenshots/30-port-security.png) |
| S13 | Guest → internet | Guest phone → 198.51.100.10 | Allowed | ✅ | [37](../screenshots/37-guest-isolation-test.png) |
| S14 | Guest → office | Guest phone → MANAGER | Blocked | ✅ | [37](../screenshots/37-guest-isolation-test.png) |
| S15 | Guest → POS | Guest phone → KITCHEN-PRN | Blocked | ✅ | [37](../screenshots/37-guest-isolation-test.png) |
| S16 | No open default Wi-Fi | AP-GUEST Port 1 | WPA2-PSK, AES | ✅ | — |

**Summary:** 32 tests, 31 passed outright and 1 passed with a documented Packet Tracer limitation.
