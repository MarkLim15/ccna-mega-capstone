# Project Screenshots

The topology diagram is included here: [`topology.png`](topology.png).

## CLI evidence record

The CLI screenshots reviewed during the capstone were used to update the verification record. The confirmed results are documented in [`verification-and-troubleshooting.md`](../verification-and-troubleshooting.md), including:

- R2-BRANCH OSPF neighbor state, learned HQ routes, and OSPF-learned default route
- R1-HQ NAT/PAT translation for Branch Guest HTTP traffic
- R2-BRANCH Guest ACL rules and inbound attachment to `GigabitEthernet0/0.40`
- SW1-HQ LACP EtherChannel status and STP root status for VLAN 99
- SSH success from PC-HQ-ADMIN and refusal from PC-HQ-USER

The repository currently stores the topology image as a separate image file; the additional CLI captures are represented by their verified findings in the documentation rather than committed as individual image files. The verification notes distinguish directly reviewed output from tests previously reported and from checks that remain unverified.
