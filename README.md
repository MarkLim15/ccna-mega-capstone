# CCNA Mega Capstone — Enterprise Network Infrastructure Lab

This is the network I built in **Cisco Packet Tracer** while studying for the CCNA. I wanted one lab where I could practise the topics together, troubleshoot real connectivity issues, and keep a record of what I changed and how I tested it.

The design connects a headquarters LAN, a branch LAN, and a simulated ISP. It brings together VLANs, inter-VLAN routing, OSPF, DHCP, DNS, NAT/PAT, access control, SSH, STP, and EtherChannel.

## Network Topology

![CCNA Mega Capstone topology](screenshots/topology.png)

The diagram shows the main devices, VLANs, HQ-to-branch connection, ISP link, and simulated public web server. The original screenshot is stored in the repository under `screenshots/topology.png`.

## What I Implemented

- VLAN segmentation at the headquarters and branch
- 802.1Q trunking and a dedicated unused native VLAN (VLAN 999)
- Router-on-a-Stick (ROAS) inter-VLAN routing
- OSPF Area 0 between R1-HQ and R2-BRANCH
- Default-route propagation from HQ toward the simulated ISP
- Central DHCP pools and DHCP relay using `ip helper-address`
- DNS records for internal and public web names
- HTTP services on the HQ server and simulated public web server
- NAT/PAT on R1-HQ for the client subnets
- An extended ACL to restrict Guest VLAN access
- SSHv2 remote management with local authentication
- Spanning Tree Protocol (STP) and an LACP EtherChannel
- NTP configuration for network devices

## VLAN Design

| Site | VLAN | Purpose | Network | Gateway |
|---|---:|---|---|---|
| HQ | 10 | Users | `192.168.10.0/24` | `192.168.10.1` |
| HQ | 20 | Admin | `192.168.20.0/24` | `192.168.20.1` |
| HQ | 99 | Management | `192.168.99.0/24` | `192.168.99.1` |
| HQ | 100 | Services | `192.168.100.0/24` | `192.168.100.1` |
| Branch | 30 | Staff | `192.168.30.0/24` | `192.168.30.1` |
| Branch | 40 | Guest | `192.168.40.0/24` | `192.168.40.1` |
| Branch | 99 | Management | `192.168.199.0/24` | `192.168.199.1` |
| Both sites | 999 | Unused native VLAN | No client subnet | None |

VLAN 99 is reused at the branch, but the HQ and branch management networks use different IP subnets because they are separate Layer 2 domains. VLAN 999 is reserved as the native VLAN on trunks and is not used for client devices.

## Routed Networks

| Connection | Interface addresses | Network |
|---|---|---|
| R1-HQ — R2-BRANCH | R1 G0/1: `10.0.12.1`; R2 G0/1: `10.0.12.2` | `10.0.12.0/30` |
| R1-HQ — R3-ISP | R1 G0/2: `198.51.100.2`; R3 G0/0: `198.51.100.1` | `198.51.100.0/30` |
| ISP public network | R3 G0/1: `203.0.113.1`; PUBLIC-WEB: `203.0.113.10` | `203.0.113.0/24` |

The full switch management addresses, server details, DHCP scopes, and DNS records are in the [IP Addressing Plan](addressing-plan.md).

## Routing Design

**R1-HQ** provides the HQ VLAN gateways and runs OSPF with **R2-BRANCH** over the `10.0.12.0/30` transit link. R1 also has a static default route toward **R3-ISP** and advertises the default into the internal OSPF domain.

**R2-BRANCH** provides the branch VLAN gateways and learns the HQ networks and default route through OSPF. The user-facing VLAN subinterfaces are passive for OSPF so they do not form neighbor adjacencies with end hosts.

**R3-ISP** represents the outside network. It connects the R1 transit link to the public `203.0.113.0/24` segment and is not part of the internal OSPF process.

## Switching Design

**SW1-HQ** and **SW2-HQ** are connected by an LACP EtherChannel using FastEthernet0/23 and FastEthernet0/24. A separate redundant Layer 2 link provides another path between the HQ switches.

STP keeps the redundant design loop-free. During troubleshooting, I had to correct the native VLAN and allowed-VLAN lists on the HQ trunks; once both ends matched, the native VLAN inconsistency cleared and the management SVIs recovered.

The branch switch **SW3-BRANCH** carries VLANs 30, 40, 99, and 999 on its trunk toward R2-BRANCH.

## Network Services

**SRV-HQ** uses `192.168.100.10/24` with gateway `192.168.100.1`. In Packet Tracer, it provides DHCP, DNS, HTTP, and NTP.

The DNS records used in testing are:

| DNS name | Address |
|---|---|
| `intranet.lab` | `192.168.100.10` |
| `www.test.net` | `203.0.113.10` |

DHCP pools provide addresses for HQ Users, HQ Admin, Branch Staff, and Branch Guest. R1 and R2 use DHCP relay to forward the client VLAN requests to SRV-HQ.

## Security — Guest VLAN ACL

R2-BRANCH's `show ip interface g0/0.40` output confirms that `GigabitEthernet0/0.40` is up/up, has helper address `192.168.100.10`, and lists `guest` as the inbound access list.

| Traffic | Policy |
|---|---|
| DHCP requests | Permit |
| DNS queries to SRV-HQ | Permit |
| HTTP to PUBLIC-WEB | Permit |
| HQ Admin subnet `192.168.20.0/24` | Deny |
| HQ Management subnet `192.168.99.0/24` | Deny |
| Branch Management subnet `192.168.199.0/24` | Deny |
| Other unmatched IP traffic | Deny |

The goal is to let guests get an address, resolve the public hostname, and browse the simulated public web server without exposing internal admin or management networks.

## SSH Remote Management

SSHv2 and local authentication are configured on the network devices. The intended management source is the HQ Admin subnet `192.168.20.0/24`.

I tested the SSH source restriction on R2 using the same destination (`10.0.12.2`) from two PCs: PC-HQ-ADMIN reached the `R2>` prompt after login, while PC-HQ-USER received `Connection refused by remote host`. This supports the intended source restriction on R2 for these two tests. The equivalent allow/deny tests have not been captured for every other network device, so I am not claiming the restriction is fully validated across all devices.

## NAT/PAT and Simulated Internet

R1-HQ performs PAT for the client subnets:

- `192.168.10.0/24` — HQ Users
- `192.168.20.0/24` — HQ Admin
- `192.168.30.0/24` — Branch Staff
- `192.168.40.0/24` — Branch Guest

The inside-global address is R1's ISP-facing address, `198.51.100.2`. In the reviewed `show ip nat translations` screenshot, Branch Guest traffic from `192.168.40.10:1025` was translated to `198.51.100.2:1025` while accessing PUBLIC-WEB at `203.0.113.10:80` over TCP/HTTP. The client was also able to load `http://www.test.net` during a separate client test.

## Verification

I used IOS show commands and end-to-end tests to check the lab instead of relying only on the running configuration.

Useful commands:

```text
show ip interface brief
show ip ospf neighbor
show ip route
show ip route 0.0.0.0
show ip nat translations
show ip nat statistics
show access-lists
show etherchannel summary
show spanning-tree vlan 99
show interfaces trunk
show ntp associations
show ip ssh
```

What I observed during the lab:

- R2's `show ip ospf neighbor` output showed neighbor ID `1.1.1.1` in `FULL/BDR` state at `10.0.12.1` on `GigabitEthernet0/1`.
- R2's `show ip route` output showed OSPF-learned HQ routes for `192.168.10.0/24`, `192.168.20.0/24`, `192.168.99.0/24`, and `192.168.100.0/24` via `10.0.12.1`.
- R2's routing table identified `10.0.12.1` as the gateway of last resort and showed `O*E2 0.0.0.0/0 [110/1]` via that next hop.
- R3 could reach R1's ISP-facing address and PUBLIC-WEB.
- The reviewed R1 `show ip nat translations` screenshot showed `192.168.40.10:1025` translated to `198.51.100.2:1025` for TCP traffic to `203.0.113.10:80`.
- The reviewed R2 `show access-lists guest` output showed DHCP, DNS, and public HTTP permits plus denies for the HQ Admin, HQ Management, and Branch Management subnets, followed by `deny ip any any`; the DHCP permit displayed five matches. The screenshot did not show an HTTP-rule hit count.
- `http://www.test.net` loaded during a client test.
- SW1-HQ's `show etherchannel summary` output showed `Po1(SU)` using LACP, with `Fa0/23(P)` and `Fa0/24(P)` bundled in the port-channel.
- SW1-HQ's `show spanning-tree vlan 99` output identified SW1-HQ as the root bridge for VLAN 99. The listed interfaces were in designated-forwarding state; this output does not identify a blocked port on a neighboring switch.
- The switch management SVIs reached `up/up` and could ping their gateways.
- SSH to R2 succeeded from PC-HQ-ADMIN; the same SSH attempt from PC-HQ-USER was refused by the remote host, supporting R2's management-source restriction for these tested sources.

The core lab has been built and the main connectivity tests described above were performed. I’m keeping two technical caveats clear rather than overstating the results: the SSH connection to R2 was tested successfully, but the SSH source restriction was not tested from both allowed and unauthorized clients on every device; and although several devices selected SRV-HQ as their NTP peer, some Packet Tracer offsets were unusually large, so accurate time synchronization is not confirmed. These are optional follow-up validation items, not missing project files.

## Evidence Screenshots

These screenshots provide supporting CLI and client-side evidence for the checks described above.

| Area | Evidence |
|---|---|
| OSPF neighbor | [OSPF_NEIGHBOR.png](screenshots/OSPF_NEIGHBOR.png) |
| Learned routes and default route | [IP_ROUTE.png](screenshots/IP_ROUTE.png) |
| NAT/PAT translation | [NAT_translation.png](screenshots/NAT_translation.png) |
| Guest ACL rules | [Access-list.png](screenshots/Access-list.png) |
| Guest ACL inbound attachment | [R2_int_g0_0.40.png](screenshots/R2_int_g0_0.40.png) |
| Public web page from Guest PC | [test_net.png](screenshots/test_net.png) |
| EtherChannel status | [sw1 etherchannel.png](screenshots/sw1%20etherchannel.png) |
| STP root status for VLAN 99 | [STP_vlan99_sw1.png](screenshots/STP_vlan99_sw1.png) |
| SSH allowed from HQ Admin | [HQ_admin_ssh.png](screenshots/HQ_admin_ssh.png) |
| SSH refused from HQ User | [HQ_User_ssh.png](screenshots/HQ_User_ssh.png) |

## Troubleshooting Notes

### 1. Native VLAN mismatch between the HQ switches

**Symptom:** STP reported a PVID inconsistency and VLAN 99 was blocked on the port-channel.

**Cause:** The native VLAN and allowed-VLAN lists did not match on both ends of the HQ trunks.

**Fix:** I created VLAN 999 and updated both sides to use it as the native VLAN, with matching allowed VLANs.

**Verification:** The inconsistency cleared, LACP remained up, and STP selected the redundant path without creating a switching loop.

### 2. Switch management SVI was down

**Symptom:** The switch management SVI had an IP address but showed `up/down`, and management pings failed.

**Checks and fix:** I checked VLAN existence, trunk settings, native VLAN consistency, allowed VLANs, STP, and neighbor information. After correcting the trunk/VLAN configuration, I bounced SW2's VLAN 99 SVI; it returned to `up/up` and management pings succeeded.

### 3. SSH to R2 failed

**Symptom:** SSH to R2 at `10.0.12.2` failed.

**Cause:** The VTY lines were not consistently configured for local authentication and SSH-only transport.

**Fix:** I enabled SSHv2 and configured VTY lines 0–15 with `login local` and `transport input ssh`.

**Verification:** `show ip ssh` reported version 2.0, and the HQ Admin PC successfully connected to R2.

### 4. NAT source ACL was too broad

**Symptom:** The NAT source ACL originally permitted any source address.

**Fix:** I narrowed it to the four client subnets listed above.

**Verification:** The NAT table showed translation for Branch Guest HTTP traffic. Each other client VLAN should be tested separately to confirm its translation.

More detail is available in [Verification and Troubleshooting](verification-and-troubleshooting.md).

## Packet Tracer File and Project Evidence

The Packet Tracer lab file is included in this repository: **[`ccna-mega-capstone.pkt`](ccna-mega-capstone.pkt)**. Download it and open it in Cisco Packet Tracer to explore the topology, inspect the configurations, and repeat the tests described here. Keep a working copy before experimenting with changes.

The topology screenshot is saved in [`screenshots/topology.png`](screenshots/topology.png), and the [IP Addressing Plan](addressing-plan.md) and [Verification and Troubleshooting](verification-and-troubleshooting.md) pages document the design and the checks performed.

The core project package is complete: the Packet Tracer file, topology image, README, addressing plan, troubleshooting notes, and supporting CLI/client screenshots are included. Separate sanitized configuration snapshots remain optional.

## What I Learned

This project helped me connect topics that felt separate when I first studied them. VLANs, trunks, ROAS, DHCP relay, OSPF, ACLs, NAT/PAT, STP, EtherChannel, and SSH all affect the same packet flow once they are part of one network.

The most useful habit was troubleshooting one layer at a time: checking the interface and VLAN state, confirming routes and next hops, looking at NAT translations, then testing the actual service from a client.

## Optional Improvements

Useful follow-up checks would be to repeat SSH source-restriction tests on the remaining devices and capture a clearer NTP association result. R2's authorized and unauthorized SSH tests are included in the evidence screenshots. The NTP caveat will remain documented unless the unusually large Packet Tracer offsets can be explained.

## Author

**Mark Anthony Lim**

Computer Engineering graduate building hands-on experience in networking and IT infrastructure.
