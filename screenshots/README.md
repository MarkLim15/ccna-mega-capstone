# Evidence Screenshots

The topology diagram is included as [`topology.png`](topology.png). The other images below are screenshots captured from the Packet Tracer lab and referenced by the main [README](../README.md) and [Verification and Troubleshooting](../verification-and-troubleshooting.md).

| File | What it shows |
|---|---|
| [OSPF_NEIGHBOR.png](OSPF_NEIGHBOR.png) | R2-BRANCH OSPF adjacency in `FULL/BDR` state |
| [IP_ROUTE.png](IP_ROUTE.png) | HQ routes learned via OSPF and OSPF external default route |
| [NAT_translation.png](NAT_translation.png) | Branch Guest TCP traffic translated by PAT toward PUBLIC-WEB |
| [Access-list.png](Access-list.png) | Configured Guest ACL permit and deny rules |
| [R2_int_g0_0.40.png](R2_int_g0_0.40.png) | Guest ACL attached inbound on R2 `GigabitEthernet0/0.40` |
| [test_net.png](test_net.png) | Guest PC browser displaying `http://www.test.net` |
| [sw1 etherchannel.png](sw1%20etherchannel.png) | SW1-HQ LACP EtherChannel member ports bundled |
| [STP_vlan99_sw1.png](STP_vlan99_sw1.png) | SW1-HQ identified as root for VLAN 99 |
| [HQ_admin_ssh.png](HQ_admin_ssh.png) | SSH login succeeds from PC-HQ-ADMIN |
| [HQ_User_ssh.png](HQ_User_ssh.png) | SSH connection is refused from PC-HQ-USER |

The verification notes distinguish directly reviewed screenshot results from tests previously reported and from checks that remain unverified.
