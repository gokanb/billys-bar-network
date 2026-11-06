# Wireless configuration (GUI-based)

The WLC and access points are configured through web pages and device tabs, not the CLI, so this file records the final settings.

> Passwords are lab-only values.

## WLC-2504 (`WLC-BILLYS`)

Managed from MANAGER → Web Browser → `https://10.10.1.117`

| Setting | Value |
|---|---|
| Management IP | 10.10.1.117 /28, gateway 10.10.1.113, VLAN 0 (untagged, lands in native VLAN 10 on CORE Fa0/20) |
| NTP server | 10.10.1.132 |
| Admin user | `admin` / `B1llyWlc$1` |
| Global DHCP proxy | Disabled |

### Dynamic interfaces

| Name | VLAN | IP | Gateway | DHCP server |
|---|---|---|---|---|
| staff | 60 | 10.10.1.2 /27 | 10.10.1.1 | 10.10.1.130 |
| guest | 70 | 10.10.0.2 /24 | 10.10.0.1 | 10.10.1.130 |

### WLANs

| ID | SSID | Security | Interface | Status |
|---|---|---|---|---|
| 1 | BILLYS-STAFF | WPA2-PSK (AES), key `Staff$Billy2026` | staff | Enabled |
| 2 | BILLYS-GUEST | WPA2-PSK (AES) | guest | **Disabled**, moved to the standalone AP-GUEST (see README, issue #1) |

### Lightweight APs (3702i)

| AP | Switch port | IP (DHCP, MGMT pool) | WLC |
|---|---|---|---|
| AP-FOH | SW-FOH Fa0/20 (access VLAN 10) | 10.10.1.119 | joined (REG) |
| AP-BAR | SW-BAR Fa0/20 (access VLAN 10) | 10.10.1.120 | joined (REG) |

Each AP got power from an `ACCESS_POINT_POWER_ADAPTER` (2960s don't supply PoE), and the WLC IP was set under AP → Config → WLC.

## AP-GUEST (AP-PT-N, standalone)

| Setting | Value |
|---|---|
| Wired port | Port 0 → SW-BAR Fa0/21 (access VLAN 70) |
| SSID | BILLYS-GUEST |
| Security | WPA2-PSK, AES, key `BillysGuest2026` |
| Result | Guests get 10.10.0.x from the GUEST-WIFI pool, and the GUEST-IN ACL limits them to the internet only |
