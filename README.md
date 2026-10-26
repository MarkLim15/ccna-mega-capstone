# CCNA Mega Capstone — Enterprise Network Infrastructure Lab

This is the network I built in **Cisco Packet Tracer** while studying for the CCNA. I wanted one lab where I could practise the topics together, troubleshoot real connectivity issues, and keep a record of what I changed and how I tested it.

The design connects a headquarters LAN, a branch LAN, and a simulated ISP. It brings together VLANs, inter-VLAN routing, OSPF, DHCP, DNS, NAT/PAT, access control, SSH, STP, and EtherChannel.

## Network Topology

![CCNA Mega Capstone topology](screenshots/topology.png)

The diagram shows the main devices, VLANs, the HQ-to-branch connection, the ISP link, and the simulated public web server. The image is from the topology screenshot uploaded for this capstone.

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

The extended ACL named `guest` is applied inbound on R2-BRANCH `G0/0.40`.

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

I verified an SSH connection from the HQ Admin PC to R2's transit address `10.0.12.2` after correcting the VTY configuration. The SSH source restriction should still be tested from both an authorized and an unauthorized source on every device before calling that policy fully validated.

## NAT/PAT and Simulated Internet

R1-HQ performs PAT for the client subnets:

- `192.168.10.0/24` — HQ Users
- `192.168.20.0/24` — HQ Admin
- `192.168.30.0/24` — Branch Staff
- `192.168.40.0/24` — Branch Guest

The inside-global address is R1's ISP-facing address, `198.51.100.2`. During testing, the NAT table showed Branch Guest traffic from `192.168.40.10` translated to `198.51.100.2` while accessing PUBLIC-WEB at `203.0.113.10` over HTTP. The client was also able to load `http://www.test.net`.

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

- R1 and R2 formed an OSPF neighbor relationship in the `FULL` state.
- R2 learned the HQ routes and the default route through OSPF.
- R3 could reach R1's ISP-facing address and PUBLIC-WEB.
- R1 showed a NAT translation for Guest HTTP traffic to the public server.
- `http://www.test.net` loaded during a client test.
- The HQ LACP EtherChannel formed, and STP handled the redundant Layer 2 path.
- The switch management SVIs reached `up/up` and could ping their gateways.
- SSH to R2 worked from the HQ Admin PC after its VTY settings were corrected.

Some items still need better evidence before I call them fully verified across all devices. In particular, I still need screenshots of the SSH allowed/denied tests and more reliable confirmation of NTP synchronization. Packet Tracer displayed SRV-HQ as the selected NTP peer on several devices, but some reported offsets were unusually large.

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

The topology screenshot is saved in [`screenshots/topology.png`](screenshots/topology.png). The `screenshots/` folder also has a checklist of evidence to capture from this capstone.

The current repository contains the project documentation and topology image. The Packet Tracer activity file and device configuration examples are not included yet; I’ll add the final lab file and sanitized configuration snapshots separately so they match this capstone and don’t expose passwords or secret hashes.

## What I Learned

This project helped me connect topics that felt separate when I first studied them. VLANs, trunks, ROAS, DHCP relay, OSPF, ACLs, NAT/PAT, STP, EtherChannel, and SSH all affect the same packet flow once they are part of one network.

The most useful habit was troubleshooting one layer at a time: checking the interface and VLAN state, confirming routes and next hops, looking at NAT translations, then testing the actual service from a client.

## Next Steps

I plan to keep using this topology to practise controlled failures and recovery. Next, I’ll add the current Packet Tracer activity file, capture the key verification outputs, and test the SSH management restriction from both allowed and disallowed sources.

## Author

**Mark Anthony Lim**

Computer Engineering graduate building hands-on experience in networking and IT infrastructure.
