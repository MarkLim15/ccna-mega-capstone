# CCNA Mega Capstone — Cisco Packet Tracer

This is the capstone lab I built while studying for the CCNA. I wanted to bring the topics I’d been practising into one network so I could see how switching, routing, network services, Internet access, and security work together.

The lab has a headquarters network, a branch network, and a simulated ISP. I also kept notes on the problems I ran into along the way and the checks I used to get things working again.

## What I built

- VLANs for users, admin, management, services, branch staff, and guests
- Router-on-a-Stick inter-VLAN routing
- OSPF Area 0 between the HQ and branch routers
- A default route toward a simulated ISP
- Central DHCP, DNS, HTTP, and NTP services
- DHCP relay for client VLANs
- NAT/PAT for the simulated public network
- A Guest ACL to limit access to internal resources
- SSHv2 remote management
- STP and an LACP EtherChannel for the redundant HQ switching path

## Devices

| Device | Purpose |
|---|---|
| R1-HQ | HQ VLAN gateways, OSPF, default route, and NAT/PAT |
| R2-BRANCH | Branch gateways, OSPF, DHCP relay, and Guest ACL |
| R3-ISP | Simulated ISP and public server network |
| SW1-HQ | HQ switching; configured as the intended STP root |
| SW2-HQ | HQ server access and redundant Layer 2 path |
| SW3-BRANCH | Branch access switching |
| SRV-HQ | Central DHCP, DNS, internal HTTP, and NTP services |
| PUBLIC-WEB | Simulated public web server |
| Client PCs | HQ User, HQ Admin, Branch Staff, and Branch Guest |

## Network overview

![Completed Cisco Packet Tracer topology](screenshots/topology.png)

*Topology screenshot captured from the current capstone lab.*

R1 connects to R2 over `10.0.12.0/30`. R1 connects to R3 over `198.51.100.0/30`. R3 provides the simulated public network `203.0.113.0/24`, which contains PUBLIC-WEB at `203.0.113.10`.

R1 and R2 form the internal OSPF Area 0 adjacency. R3 is the simulated ISP and is not part of the internal OSPF process.

## VLANs and addressing

| Site | VLAN | Purpose | Subnet | Gateway |
|---|---:|---|---|---|
| HQ | 10 | Users | `192.168.10.0/24` | `192.168.10.1` |
| HQ | 20 | Admin | `192.168.20.0/24` | `192.168.20.1` |
| HQ | 99 | Management | `192.168.99.0/24` | `192.168.99.1` |
| HQ | 100 | Services | `192.168.100.0/24` | `192.168.100.1` |
| Branch | 30 | Staff | `192.168.30.0/24` | `192.168.30.1` |
| Branch | 40 | Guest | `192.168.40.0/24` | `192.168.40.1` |
| Branch | 99 | Management | `192.168.199.0/24` | `192.168.199.1` |
| Both sites | 999 | Unused native VLAN | No client subnet | None |

VLAN 99 is reused at the branch, but HQ and branch management use different IP subnets because they are separate Layer 2 domains. VLAN 999 is reserved as the unused native VLAN on trunks.

Switch management addresses:
- SW1-HQ: `192.168.99.2/24`
- SW2-HQ: `192.168.99.3/24`
- SW3-BRANCH: `192.168.199.2/24`

The full details are in [the addressing plan](addressing-plan.md).

## Network services and Internet access

SRV-HQ is `192.168.100.10/24`, with default gateway `192.168.100.1`. It provides the central DHCP, DNS, HTTP, and NTP services in Packet Tracer.

The DNS records used for client tests are:
- `intranet.lab` → `192.168.100.10`
- `www.test.net` → `203.0.113.10`

PUBLIC-WEB is `203.0.113.10/24`, with gateway `203.0.113.1`. R1 performs PAT for the client subnets, translating outbound traffic to its ISP-facing address `198.51.100.2`.

## Guest access rules

The extended ACL named `guest` is applied inbound on R2's `G0/0.40` Guest subinterface. It allows DHCP, DNS queries to SRV-HQ, and HTTP to PUBLIC-WEB. It denies access to the HQ Admin subnet, HQ management subnet, branch management subnet, and other unmatched IP traffic.

## What I verified

During testing, I observed the R1–R2 OSPF neighbor reach `FULL`, confirmed that the routers learned the expected remote networks, and verified that R2 learned the default route through OSPF. R1 showed NAT translations for Branch Guest traffic to the public web server, and `http://www.test.net` loaded in a client test.

The LACP EtherChannel formed, STP handled the redundant Layer 2 path, and the switch management SVIs reached `up/up`. I also fixed R2's SSH VTY setup and confirmed that an SSH connection from the HQ Admin PC worked.

Some checks need stronger evidence before I treat them as fully verified on every device. In particular, the SSH source restriction should be tested from both an allowed and an unauthorized client. The NTP server was selected as a peer on several devices, but some Packet Tracer offsets were unusually large, so precise clock synchronization still needs further checking.

The [verification and troubleshooting notes](verification-and-troubleshooting.md) record the tests and remaining checks.

## Files to add

- [ ] Final Packet Tracer activity file
- [ ] Screenshot of the completed topology
- [ ] OSPF neighbor and routing-table evidence
- [ ] NAT/PAT translation evidence
- [ ] Guest ACL allowed/blocked traffic tests
- [ ] EtherChannel and STP verification
- [ ] SSH allowed/denied access tests

I’ll add these items as I capture the final evidence from the current capstone. I won’t reuse screenshots from my older enterprise-network project because it has a different topology and addressing plan.

## What I learned

The main lesson for me was that a configured feature isn’t automatically a working feature. A trunk can be up but allow the wrong VLANs, an SVI can have an IP address but remain down, and an SSH login can fail because of the VTY configuration even when a user account exists.

I tried to troubleshoot one layer at a time: check the relevant IOS output, make a targeted change, and test again.

## Author

**Mark Anthony Lim**

Computer Engineering graduate building hands-on experience in networking and IT infrastructure.
