# IP Addressing Plan

This page records the address plan used in my CCNA Mega Capstone Packet Tracer lab.

## Routed links

| Connection | Device/interface | Address |
|---|---|---|
| HQ–Branch transit | R1 G0/1 | `10.0.12.1/30` |
| HQ–Branch transit | R2 G0/1 | `10.0.12.2/30` |
| HQ–ISP transit | R1 G0/2 | `198.51.100.2/30` |
| HQ–ISP transit | R3 G0/0 | `198.51.100.1/30` |
| ISP public LAN | R3 G0/1 | `203.0.113.1/24` |
| Public web server | PUBLIC-WEB | `203.0.113.10/24`, gateway `203.0.113.1` |

## HQ VLANs

| VLAN | Purpose | Subnet | Gateway |
|---:|---|---|---|
| 10 | Users | `192.168.10.0/24` | `192.168.10.1` on R1 G0/0.10 |
| 20 | Admin | `192.168.20.0/24` | `192.168.20.1` on R1 G0/0.20 |
| 99 | Management | `192.168.99.0/24` | `192.168.99.1` on R1 G0/0.99 |
| 100 | Services | `192.168.100.0/24` | `192.168.100.1` on R1 G0/0.100 |
| 999 | Unused native VLAN | No client subnet | None |

## Branch VLANs

| VLAN | Purpose | Subnet | Gateway |
|---:|---|---|---|
| 30 | Staff | `192.168.30.0/24` | `192.168.30.1` on R2 G0/0.30 |
| 40 | Guest | `192.168.40.0/24` | `192.168.40.1` on R2 G0/0.40 |
| 99 | Management | `192.168.199.0/24` | `192.168.199.1` on R2 G0/0.99 |
| 999 | Unused native VLAN | No client subnet | None |

## Switch management

| Switch | SVI | Address | Default gateway |
|---|---|---|---|
| SW1-HQ | VLAN 99 | `192.168.99.2/24` | `192.168.99.1` |
| SW2-HQ | VLAN 99 | `192.168.99.3/24` | `192.168.99.1` |
| SW3-BRANCH | VLAN 99 | `192.168.199.2/24` | `192.168.199.1` |

## Central services

| Service/device | Address | Notes |
|---|---|---|
| SRV-HQ | `192.168.100.10/24` | Gateway `192.168.100.1`; DNS points to itself |
| PUBLIC-WEB | `203.0.113.10/24` | Gateway `203.0.113.1`; HTTP enabled |

## DHCP scopes

| Scope | Network | Default gateway | DNS |
|---|---|---|---|
| HQ-Users | `192.168.10.0/24` | `192.168.10.1` | `192.168.100.10` |
| HQ-Admin | `192.168.20.0/24` | `192.168.20.1` | `192.168.100.10` |
| Branch-Staff | `192.168.30.0/24` | `192.168.30.1` | `192.168.100.10` |
| Branch-Guest | `192.168.40.0/24` | `192.168.40.1` | `192.168.100.10` |

DHCP relay on the client VLAN subinterfaces points to `192.168.100.10`.

## Trunk policy

- HQ trunks: VLANs `10,20,99,100,999`
- Branch trunk: VLANs `30,40,99,999`
- Native VLAN: `999` on each trunk
- VLAN 999 is reserved and should not carry client traffic.

## Notes

The logical address plan is documented here. For the final port-to-port physical link map, I’ll use the cabling shown in the final Packet Tracer file/screenshot rather than assume interface mappings from a draft topology.
