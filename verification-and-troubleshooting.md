# Verification and Troubleshooting Notes

I kept these notes while building and checking the capstone. They record what I actually tested and flag items that still need more evidence.

## Verification summary

| Feature | Evidence / test | Status |
|---|---|---|
| R1–R2 OSPF | `show ip ospf neighbor` showed the neighbor in `FULL` state | Verified |
| Internal routes | R1 learned Branch Staff and Guest routes; R2 learned HQ networks | Verified |
| Default route | R2 showed an OSPF external default route via R1 | Verified |
| R1 to ISP | R3 pinged `198.51.100.2` successfully | Verified |
| Public server network | R3 pinged `203.0.113.10` successfully | Verified |
| NAT/PAT | R1 translation showed `192.168.40.10` translated to `198.51.100.2` for HTTP to `203.0.113.10:80` | Verified for Branch Guest |
| DNS/HTTP | `http://www.test.net` loaded from a client during testing | Verified for tested client |
| Guest ACL placement | `guest` applied inbound to R2 `G0/0.40` | Verified |
| Guest ACL policy | User reported the allowed and blocked traffic tests passed | Reported passed; add screenshots as evidence |
| LACP | `show etherchannel summary` showed `Po1(SU)` and Fa0/23 and Fa0/24 bundled | Verified |
| HQ STP | SW1 was root for HQ VLANs; redundant path blocked where appropriate | Verified |
| Switch management SVIs | SW1 `192.168.99.2`, SW2 `192.168.99.3`, SW3 `192.168.199.2` reached `up/up` | Verified |
| SSH to R2 | Admin PC successfully connected to `10.0.12.2` after VTY corrections | Verified for R2 |
| SSH source restriction | Management-source ACL was configured on device VTY lines | Needs allowed/denied source tests captured |
| NTP | Devices pointed to `192.168.100.10`; several showed it as `* sys.peer` | Partially verified; see note below |

## Troubleshooting notes

### Native VLAN mismatch between SW1 and SW2

**Symptom:** STP reported `PVID_Inc`, and the Port-channel was blocked for VLAN 99.

**Cause:** SW1 and SW2 initially used different native VLANs and incomplete allowed-VLAN lists.

**Fix:** Both ends were configured to use native VLAN 999 and allow VLANs 10, 20, 99, 100, and 999. VLAN 999 was also created in the VLAN database.

**Verification:** The `PVID_Inc` condition cleared. LACP remained up, and STP handled the redundant path without creating a loop.

### SW1/SW2 management SVI down

**Symptom:** The management SVI had an IP address but showed `up/down`, and management pings failed.

**Checks:** VLAN 99 existence, trunk native VLAN, trunk allowed VLANs, STP state, and CDP neighbors were inspected.

**Fix and recovery:** After correcting VLAN/trunk settings, SW2's VLAN 99 SVI was shut down and re-enabled. It returned to `up/up`; ping to its gateway and to SW1's management address then succeeded.

### R2 SSH connection failed

**Symptom:** SSH to R2's transit address `10.0.12.2` failed while other tested devices were reachable.

**Cause:** R2's VTY lines were not consistently configured for local authentication and SSH-only transport.

**Fix:** VTY lines 0–15 were configured with `login local` and `transport input ssh`, and SSHv2 was enabled.

**Verification:** `show ip ssh` reported version 2.0, and the Admin PC successfully connected to `10.0.12.2`.

### NAT/PAT source scope

**Symptom:** The NAT source ACL originally permitted any source address.

**Fix:** The standard ACL was narrowed to the four client subnets: `192.168.10.0/24`, `192.168.20.0/24`, `192.168.30.0/24`, and `192.168.40.0/24`.

**Verification:** The NAT table showed a translation for Branch Guest HTTP traffic. Each of the other client VLANs should be tested separately to confirm its translation after the ACL change.

## NTP caveat

The switches were pointed to `192.168.100.10`. Several devices displayed the server as `* sys.peer`, but some Packet Tracer outputs showed an unusually large offset. I’ve recorded the server association as observed, but I’m not claiming accurate clock synchronization on every device until the offset is understood or Packet Tracer’s behavior is confirmed.

## Screenshots and final evidence to add

- [ ] Full Packet Tracer topology with device labels visible
- [ ] `show ip ospf neighbor` from R1 or R2
- [ ] `show ip route` from R2 showing the default route
- [ ] `show ip nat translations` on R1 during a public website test
- [ ] Guest PC: public website succeeds
- [ ] Guest PC: access to protected internal subnets fails
- [ ] Admin PC SSH success and an unauthorized source SSH failure
- [ ] `show etherchannel summary` and `show spanning-tree vlan 99`
- [ ] NTP association outputs after devices have had time to synchronize

I’ll capture the evidence from this capstone’s actual Packet Tracer file so these screenshots match this project, not my older lab.
