# Verification and Troubleshooting Notes

I kept these notes while building and checking the capstone. They record the tests performed and separate confirmed results from limitations that would benefit from further validation.

## Verification summary

| Feature | Evidence / test | Status |
|---|---|---|
| R1–R2 OSPF | R2 `show ip ospf neighbor` showed neighbor ID `1.1.1.1`, state `FULL/BDR`, neighbor address `10.0.12.1`, interface `GigabitEthernet0/1` | Verified — CLI screenshot reviewed |
| Internal routes | R2 `show ip route` showed OSPF routes for HQ Users (`192.168.10.0/24`), HQ Admin (`192.168.20.0/24`), HQ Management (`192.168.99.0/24`), and HQ Services (`192.168.100.0/24`), all via `10.0.12.1`; Branch Staff and Guest networks are directly connected | Verified — CLI screenshot reviewed |
| Default route | R2 showed gateway of last resort `10.0.12.1` and `O*E2 0.0.0.0/0 [110/1]` via `10.0.12.1` | Verified — CLI screenshot reviewed |
| R1 to ISP | R3 pinged `198.51.100.2` successfully | Verified |
| Public server network | R3 pinged `203.0.113.10` successfully | Verified |
| NAT/PAT | R1 `show ip nat translations` screenshot showed TCP `192.168.40.10:1025` translated to `198.51.100.2:1025` for `203.0.113.10:80` | Verified for Branch Guest — CLI screenshot reviewed |
| DNS/HTTP | `http://www.test.net` loaded from a client during testing | Verified for tested client |
| Guest ACL placement | `guest` ACL is configured inbound on R2 `G0/0.40`; the reviewed `show access-lists guest` capture displays its rule set | Verified configuration; this output alone does not display the interface attachment |
| Guest ACL policy | R2 `show access-lists guest` screenshot showed DHCP, DNS, and public HTTP permit entries, denies for HQ Admin and management subnets, and a final `deny ip any any`; DHCP permit showed 5 matches | ACL configuration reviewed; the capture does not show an HTTP-rule hit count. Allowed/blocked end-to-end tests were previously reported by the user |
| LACP | `show etherchannel summary` showed `Po1(SU)` and Fa0/23 and Fa0/24 bundled | Verified |
| HQ STP | SW1 was root for HQ VLANs; redundant path blocked where appropriate | Verified |
| Switch management SVIs | SW1 `192.168.99.2`, SW2 `192.168.99.3`, SW3 `192.168.199.2` reached `up/up` | Verified |
| SSH to R2 | Admin PC successfully connected to `10.0.12.2` after VTY corrections | Verified for R2 |
| SSH source restriction | Management-source ACL was configured on device VTY lines | Configured; a separate allowed/denied source test on every device is recommended |
| NTP | Devices pointed to `192.168.100.10`; several showed it as `* sys.peer` | Server association observed; precise time synchronization is unconfirmed because of unusual Packet Tracer offsets |

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

## Optional supporting evidence

The core deliverables are included in the repository: the [Packet Tracer file](ccna-mega-capstone.pkt), [topology screenshot](screenshots/topology.png), addressing plan, and this troubleshooting record. The following evidence would be useful if I want to expand the portfolio further, but it is not a prerequisite for using or understanding the project:

- `show ip ospf neighbor` and `show ip route` from R1/R2
- Additional NAT/PAT tests for HQ Users, HQ Admin, and Branch Staff (Branch Guest translation is now evidenced by a reviewed CLI screenshot)
- A Guest PC screenshot showing public HTTP succeeds and protected internal access is blocked; the CLI ACL screenshot documents the configured rules, but not the HTTP-rule hit count
- `show etherchannel summary` and `show spanning-tree vlan 99`
- SSH tests from an authorized source and an unauthorized source
- NTP association output after allowing time to synchronize

Any added screenshots should come from the included `ccna-mega-capstone.pkt` file so they match this project.
