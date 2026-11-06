# Server configuration (GUI-based)

All servers are Packet Tracer Server-PT devices, configured in the Desktop and Services tabs.

## Addressing

| Server | Role | IP / mask | Gateway | DNS |
|---|---|---|---|---|
| SRV-CORE | DHCP + DNS | 10.10.1.130 /29 | 10.10.1.129 | 10.10.1.130 |
| SRV-FILE | FTP file share | 10.10.1.131 /29 | 10.10.1.129 | 10.10.1.130 |
| SRV-MGMT | Syslog + NTP | 10.10.1.132 /29 | 10.10.1.129 | 10.10.1.130 |
| NVR | Camera recorder | 10.10.1.98 /28 | 10.10.1.97 | 10.10.1.130 |
| DMZ-WEB | Public website | 10.10.1.138 /29 (public 203.0.113.3) | 10.10.1.137 | none |
| DMZ-MAIL | Public mail | 10.10.1.139 /29 (public 203.0.113.4) | 10.10.1.137 | none |
| WEB-INTERNET | Fake internet site | 198.51.100.10 /24 | 198.51.100.1 | none |

## Services left running (least functionality)

| Server | ON | OFF |
|---|---|---|
| SRV-CORE | DHCP, DNS | HTTP, HTTPS, FTP, SMTP, POP3 |
| SRV-FILE | FTP | HTTP, HTTPS, SMTP, POP3, DNS |
| SRV-MGMT | SYSLOG, NTP | HTTP, HTTPS, FTP, SMTP, POP3 |
| DMZ-WEB | HTTP (HTTPS on but not published) | FTP, SMTP, POP3 |

## DHCP pools (SRV-CORE)

| Pool | Gateway | Start IP | Mask | Max users | WLC address |
|---|---|---|---|---|---|
| MGMT | 10.10.1.113 | 10.10.1.118 | 255.255.255.240 | 8 | 10.10.1.117 |
| POS | 10.10.1.33 | 10.10.1.40 | 255.255.255.224 | 20 | 0.0.0.0 |
| STAFF | 10.10.1.65 | 10.10.1.70 | 255.255.255.224 | 20 | 0.0.0.0 |
| CCTV | 10.10.1.97 | 10.10.1.100 | 255.255.255.240 | 10 | 0.0.0.0 |
| STAFF-WIFI | 10.10.1.1 | 10.10.1.10 | 255.255.255.224 | 20 | 0.0.0.0 |
| GUEST-WIFI | 10.10.0.1 | 10.10.0.10 | 255.255.255.0 | 240 | 0.0.0.0 |

All pools use DNS 10.10.1.130. Pools start above the static addresses (printers, NVR, WLC). The MGMT pool's **WLC address** (DHCP option 43) is how the lightweight APs find the controller.

## DNS A records (SRV-CORE)

| Name | Address |
|---|---|
| srv-core.billysbar.local | 10.10.1.130 |
| files.billysbar.local | 10.10.1.131 |
| mgmt.billysbar.local | 10.10.1.132 |
| nvr.billysbar.local | 10.10.1.98 |
| core.billysbar.local | 10.10.1.113 |

## FTP users (SRV-FILE)

The default `cisco/cisco` account was removed.

| User | Permissions | Why |
|---|---|---|
| manager | Write, Read, Delete, Rename, List | Uploads menus, schedules, invoices |
| staff | Read, List | View/download only (least privilege) |

## Static end devices

| Device | IP | Gateway | Where set |
|---|---|---|---|
| KITCHEN-PRN | 10.10.1.34 /27 | 10.10.1.33 | Config → FastEthernet0 + Settings |
| BAR-PRN | 10.10.1.35 /27 | 10.10.1.33 | same |
| OFFICE-PRN | 10.10.1.66 /27 | 10.10.1.65 | same |
