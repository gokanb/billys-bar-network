# Packet Tracer files

Built in **Cisco Packet Tracer 9.0.1**. Open any file with PT 9.0.1 or newer.

| File | Checkpoint |
|---|---|
| `01-cabling.pkt` | Devices placed, renamed and cabled; VLAN work started on CORE |
| `02-vlans-trunks.pkt` | VLANs on all switches, access ports, trunks (native VLAN 99) |
| `03-intervlan-routing.pkt` | `ip routing`, SVIs on CORE, static server IPs |
| `04-dhcp.pkt` | DHCP pools, `ip helper-address`, client + static addressing |
| `05-services.pkt` | DNS, FTP, switch management IPs, Syslog, NTP |
| `06-wireless.pkt` | WLC + lightweight APs, staff/guest WLANs |
| `07-firewall-dmz.pkt` | ISP, ASA zones, NAT/PAT, DMZ with public website |
| `08-security.pkt` | SSH, blackhole VLAN, port security, ACLs, hardening |
| **`billys-bar-FINAL.pkt`** | **Finished network, including the working isolated guest Wi-Fi** |

Each file is a snapshot saved at a checkpoint, so a few of them already include the first part of the next step.

**Lab credentials** (lab-only values): console `B1llys$Con` → enable `B1llys$Enable` · SSH `admin` / `B1llys$Admin` · ASA enable `B1llys$FW` · WLC `admin` / `B1llyWlc$1`
