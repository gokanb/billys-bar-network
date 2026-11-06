# Troubleshooting log

These are the real problems I hit while building this, how I tracked each one down, and what fixed it. Honestly, this is where I learned the most.

---

## 1. Cameras won't answer pings from other VLANs

**Symptom:** MANAGER (VLAN 30) could ping the NVR (VLAN 50) but not the webcams, even though they're on the same switch and the same VLAN as the NVR.

**What I checked:**
- Camera IP, mask, and gateway were all correct (10.10.1.101/28, gw .97)
- The NVR could ping the camera, so the camera itself was up
- In Simulation mode, the packet reached the camera but no reply came back out

**Conclusion:** This is a Packet Tracer IoT webcam limitation. They don't reply to cross-VLAN ICMP. Same-VLAN camera ↔ NVR traffic works, and that's the only traffic my design allows anyway (cameras → NVR only).

---

## 2. Internet web pages timed out, but ping worked

**Symptom:** `ping 198.51.100.10` from MANAGER worked (TTL 125), but `http://198.51.100.10` timed out.

**What I tried:** confirmed HTTP was on, added `inspect http`, and temporarily added an ACL permitting `tcp any eq www any`. None of these made a difference.

**How I found it:** I ran a Simulation trace filtered to HTTP + TCP (screenshot 20). The first SYN died at the FW, the retry died at the ISP, and the third one made it all the way. Then the HTTP request and response went through fine.

**Root cause:** ARP warm-up. A brand-new 5-hop path (MANAGER → SW-BOH → CORE → FW → ISP → server) needs every device to learn MAC addresses first, and Packet Tracer's browser gave up before that finished. Once the path was warm, browsing worked.

**Cleanup:** I removed the test ACL line, because a rule you don't need is a rule you shouldn't keep. Browsing still worked after.

---

## 3. ASA doesn't support port forwarding in Packet Tracer

**Symptom:** WEB-INTERNET got "Server Reset Connection" when it opened `http://203.0.113.2`.

**What I found:** The caret pointed at `service` in:
```
nat (dmz,outside) static interface service tcp www www
                                   ^ Invalid input
```
So the port forward was never created (screenshot 23). A real ASA supports this, but the Packet Tracer ASA doesn't.

**Fix:** I did what real businesses do and asked the ISP for a bigger block. I moved the outside link from /30 to **/29** (203.0.113.0/29), then gave each DMZ server its own public IP with 1-to-1 static NAT:
- 203.0.113.3 → DMZ-WEB
- 203.0.113.4 → DMZ-MAIL

This also required changing the ISP's G0/0 mask to /29. Until I did, the ISP treated .3 as the broadcast address.

---

## 4. Typo in a firewall rule

**Symptom:** Found during review: the SMTP rule pointed to **10.10.1.138** (the web server) instead of **.139** (the mail server).

**Impact:** The web server had an unnecessary open port, and the mail server was unreachable from outside.

**Fix:** I removed the wrong line, added the correct one, and verified with `show access-list`. The hit counter (`hitcnt=1`) on the www rule confirmed the website rule was the one being used (screenshot 27).

**Lesson:** Always run a show command after a firewall change. One wrong digit opens the wrong door.

---

## 5. Commands silently rejected on the ASA

**Symptom:** Pasted config "didn't work."

**Cause:** The ASA drops back to `FW>` (user mode) every time its window is reopened. Everything pasted at `>` gets rejected with `% Invalid input`.

**Fix/habit:** Check the prompt first. `FW#` is needed to run commands, and `FW(config)#` to configure. Also, `exit` from a sub-mode can take you all the way out of config mode, which left `service-policy` running at the wrong level once.

---

## 6. Guest Wi-Fi users landed in the management VLAN (the big one)

**Symptom:** A phone on BILLYS-GUEST (via the WLC) got **10.10.1.12x/28**, an address from the **MGMT** DHCP pool, instead of 10.10.0.x (screenshot 34). That's a serious security problem: guests sitting next to the switches and WLC.

**Everything I checked or tried:**

| Check / attempt | Result |
|---|---|
| WLAN 2 interface = `guest` | Correct |
| AP Group `default-group` mapping (BILLYS-GUEST → guest) | Correct |
| WLC dynamic interfaces (staff VLAN 60, guest VLAN 70) | Created manually; the setup wizard hadn't created them |
| Global DHCP proxy | Already disabled; per-interface option is greyed out in PT |
| AP switch ports as trunks (native 10, allow 10,60,70) | No change, reverted |
| Rebooted both APs (pulled the power adapter) | No change |
| Removed the DHCP server from the guest interface | No change, restored |
| MONITOR → Clients on the WLC | Not supported in PT |

**Workaround attempt #1:** I disabled BILLYS-GUEST on the WLC and added a standalone guest AP (`AP-PT`) on SW-BAR Fa0/21 in **VLAN 70**. Phones still wouldn't associate, even with security disabled, and `show mac address-table interface fa0/21` was empty (screenshot 35). Nothing was passing through that AP at all.

**Final fix:** I swapped the plain **AP-PT** for an **AP-PT-N** (802.11n, matching the phones' wireless cards). Every client associated immediately, and the phone got **10.10.0.13**, gw 10.10.0.1 (screenshot 36).

**Verified isolation (screenshot 37):** from the guest phone:
- internet `198.51.100.10` → replies
- MANAGER `10.10.1.70` → unreachable (GUEST-IN ACL)
- kitchen printer `10.10.1.34` → unreachable

**Lessons:**
- In Packet Tracer, the device model matters. An old AP-PT won't talk to 802.11n clients.
- A brand-new AP-PT broadcasts an **open SSID called "Default"**, and devices joined it instantly. That's a good demo of why you never leave equipment on factory settings.
- In a real job, I'd have **disabled guest Wi-Fi until it was fixed**, rather than leave guests in the management VLAN.

---

## Small things worth remembering

- **Prompt `>` vs `#`:** `copy running-config startup-config` fails at `CORE>`. You need `enable` first.
- **Setup wizard:** always answer `no` to "Continue with configuration dialog?"
- **Spanning tree:** after changing a port to a trunk, wait about 30 seconds before testing (it blocks while it checks for loops).
- **TTL tells the hop count:** 128 = same VLAN, 127 = routed once (CORE), 125 = CORE + FW + ISP.
- **First ping timeout = ARP**, not a failure.
- **Printers have two config pages:** Settings (gateway/DNS) vs FastEthernet0 (own IP). I put the printer's own IP in the gateway box once.
- **Port security CurrentAddr = 0** until the device actually sends traffic.
