# eBGP + OSPF Lab: VoIP, Inter-VLAN, Wireless AP & ACL

Two Autonomous Systems connected with **eBGP**, each running **OSPF** internally. The customer side adds router-on-a-stick inter-VLAN routing, a wireless access point, Cisco IP telephony, and an ACL. Built and tested in **Cisco Packet Tracer**.

---

## Preview

**1. Topology**
<p align="center"><img src="images/topology.png" alt="Network topology" width="900"></p>
<!-- replace with a full-topology screenshot -->

**2. eBGP and OSPF verification**
<p align="center"><img src="images/bgp-ospf-verification.png" alt="show ip bgp summary and show ip route" width="900"></p>
<!-- replace with show ip bgp summary / show ip route output -->

**3. VoIP and ACL tests**
<p align="center"><img src="images/voip-acl-test.png" alt="IP phone registration and ACL test" width="900"></p>
<!-- replace with IP phone call, ping blocked by ACL, or tracert -->

---

## Overview

| Feature | Where | Summary |
|---|---|---|
| eBGP | Router4 ↔ Router0 | AS 65001 (customer) ↔ AS 65000 (ISP) over `200.0.0.0/30` |
| OSPF | Inside both AS | Area 0; BGP routes redistributed into the ISP's OSPF |
| Inter-VLAN | Router10, Router5, Router4, Router11 | Router-on-a-stick with 802.1Q subinterfaces |
| Wireless AP | Router10, VLAN 13 | Access Point + smartphones on `192.168.13.0/24` |
| VoIP | Router11 | Cisco CME (`telephony-service`) with DHCP option 150 |
| ACL | Router10 | Blocks smartphones from Server0 |

| AS | Role | Edge router | Other routers |
|---|---|---|---|
| 65001 | Customer | Router4 | Router10 (2811), Router5 |
| 65000 | ISP | Router0 | Router6, Router11 (2811), Router8 |

---

## Addressing

### Point-to-point links

| Link | Subnet | Side A | Side B |
|---|---|---|---|
| Router10 ↔ Router5 | 10.10.10.0/30 | Router10 Se1/0 `.1` | Router5 Se0/1/0 `.2` |
| Router10 ↔ Router4 | 10.10.11.0/30 | Router10 Se1/1 `.1` | Router4 Se0/1/0 `.2` |
| **Router0 ↔ Router4 (eBGP)** | 200.0.0.0/30 | Router0 Gi0/0/0 `.1` | Router4 Gi0/0/0 `.2` |
| Router0 ↔ Router6 | 12.10.12.0/30 | Router0 Se0/1/0 `.1` | Router6 Se0/1/0 `.2` |
| Router6 ↔ Router11 | 12.10.10.0/30 | Router6 Se0/2/1 `.1` | Router11 `.2` |
| Router11 ↔ Router8 | 12.10.11.0/30 | Router11 `.1` | Router8 `.2` |
| Router6 ↔ Router8 | 12.10.13.0/30 | Router8 `.1` | Router6 Se0/2/0 `.2` |

### VLANs and subnets

| Router | Interface | VLAN | Subnet / Gateway | Devices |
|---|---|---|---|---|
| Router10 | Fa0/0 | 1 (native) | 192.168.10.0/24 → `.1` | n/a |
| Router10 | Fa0/0.11 | 11 | 192.168.11.0/24 → `.1` | IP phones |
| Router10 | Fa0/0.12 | 12 | 192.168.12.0/24 → `.1` | PC0, PC1 |
| Router10 | Fa0/0.13 | 13 | 192.168.13.0/24 → `.1` | Access Point, smartphones |
| Router5 | Gi0/0/0.15 | 15 | 100.10.10.0/24 → `.1` | Server0 (`100.10.10.2`) |
| Router4 | Gi0/0/1.21 | 21 (BLUD) | 192.168.21.0/24 → `.1` | PC2, Printer1 |
| Router4 | Gi0/0/1.22 | 22 (PR) | 192.168.22.0/24 → `.1` | PC3 |
| Router6 | Gi0/0/0.15 | 15 | 192.10.15.0/24 → `.1` | ISP PCs (DHCP) |
| Router11 | Fa0/0.10 | 10 | 192.10.10.0/24 → `.1` | ISP LAN (DHCP) |
| Router11 | Fa0/0.50 | 50 | 192.10.51.0/24 → `.1` | IP phones (VoIP) |
| Router11 | Fa0/0.52 | 52 | 192.10.52.0/24 → `.1` | PCs |
| Router8 | Gi0/0/0.30 | 30 | 100.10.30.0/24 → `.1` | Server1 (`100.10.30.2`) |

---

## eBGP

Router4 and Router0 are the only routers in different AS, so the link between them is the only eBGP session.

| Router | AS | Neighbor | Advertised prefixes |
|---|---|---|---|
| Router4 | 65001 | 200.0.0.1 (AS 65000) | 100.10.10.0/24, 192.168.10/11/12/13/21/22.0/24 |
| Router0 | 65000 | 200.0.0.2 (AS 65001) | 192.10.10.0/24, 192.10.15.0/24, 100.10.30.0/24 |

<details>
<summary><b>Router4 (customer edge)</b></summary>

```text
interface GigabitEthernet0/0/0
 ip address 200.0.0.2 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 200.0.0.1
!
router bgp 65001
 bgp router-id 4.4.4.4
 neighbor 200.0.0.1 remote-as 65000
 network 100.10.10.0 mask 255.255.255.0
 network 192.168.10.0 mask 255.255.255.0
 network 192.168.11.0 mask 255.255.255.0
 network 192.168.12.0 mask 255.255.255.0
 network 192.168.13.0 mask 255.255.255.0
 network 192.168.21.0 mask 255.255.255.0
 network 192.168.22.0 mask 255.255.255.0
```
</details>

<details>
<summary><b>Router0 (ISP edge)</b></summary>

```text
interface GigabitEthernet0/0/0
 ip address 200.0.0.1 255.255.255.252
 no shutdown
!
interface Serial0/1/0
 ip address 12.10.12.1 255.255.255.252
 no shutdown
!
router bgp 65000
 neighbor 200.0.0.2 remote-as 65001
 network 192.10.10.0 mask 255.255.255.0
 network 192.10.15.0 mask 255.255.255.0
 network 100.10.30.0 mask 255.255.255.0
```
</details>

| Rule | Why |
|---|---|
| `network` needs an exact match in the routing table | BGP only advertises prefixes it already has, with the same mask |
| The eBGP link is not placed in OSPF | It is the boundary between the two AS |
| Both ends need `no shutdown` and an IP before the session comes up | Check with `ping 200.0.0.x` first |

---

## OSPF

Process 1, area 0, in both AS. Router4 and Router0 sit on the AS boundary and add one extra command each.

| Router | OSPF networks | Extra |
|---|---|---|
| Router10 | 10.10.10.0/30, 10.10.11.0/30, 192.168.10.0/24 to 192.168.13.0/24 | Router ID `2.2.2.2` |
| Router5 | 10.10.10.0/30, 100.10.10.0/24 | Router ID `3.3.3.3` |
| Router4 | 10.10.11.0/30, 192.168.21.0/24, 192.168.22.0/24 | `default-information originate`, Router ID `4.4.4.4` |
| Router0 | 12.10.12.0/30 | `redistribute bgp 65000 subnets` |
| Router6 | 12.10.12.0/30, 12.10.10.0/30, 12.10.13.0/30, 192.10.15.0/24 | n/a |
| Router11 | 12.10.10.0/30, 12.10.11.0/30, 192.10.10.0/24 | n/a |
| Router8 | 12.10.13.0/30, 12.10.11.0/30, 100.10.30.0/24 | n/a |

<details>
<summary><b>Router4 and Router0 OSPF config</b></summary>

```text
! Router4
router ospf 1
 router-id 4.4.4.4
 passive-interface GigabitEthernet0/0/1.21
 passive-interface GigabitEthernet0/0/1.22
 network 10.10.11.0 0.0.0.3 area 0
 network 192.168.21.0 0.0.0.255 area 0
 network 192.168.22.0 0.0.0.255 area 0
 default-information originate

! Router0
router ospf 1
 network 12.10.12.0 0.0.0.3 area 0
 redistribute bgp 65000 subnets
```
</details>

| Mechanism | Purpose |
|---|---|
| `default-information originate` (Router4) | Gives Router10 and Router5 a default route toward the ISP |
| `redistribute bgp` (Router0) | Teaches the ISP routers the customer prefixes so replies have a return path |
| `passive-interface` | Stops OSPF hellos on LAN-facing interfaces |

---

## Inter-VLAN (router-on-a-stick)

One physical interface carries several VLANs through subinterfaces. The switch port facing the router is a **trunk**. Each device uses the IP of its own VLAN's subinterface as gateway.

<details>
<summary><b>Router10 subinterfaces</b></summary>

```text
interface FastEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
!
interface FastEthernet0/0.11
 encapsulation dot1Q 11
 ip address 192.168.11.1 255.255.255.0
!
interface FastEthernet0/0.12
 encapsulation dot1Q 12
 ip address 192.168.12.1 255.255.255.0
!
interface FastEthernet0/0.13
 encapsulation dot1Q 13
 ip address 192.168.13.1 255.255.255.0
```
</details>

<details>
<summary><b>Switch</b></summary>

```text
vlan 11
 name IP-PHONE
vlan 12
 name PC
vlan 13
 name AP
!
interface GigabitEthernet0/1
 switchport mode trunk
!
interface range FastEthernet0/2 - 4
 switchport mode access
 switchport access vlan 11
```
</details>

| Common mistake | Result |
|---|---|
| Gateway set to the physical interface IP | Host cannot reach its gateway |
| Router-facing port left as access | Subinterfaces never receive tagged frames |
| VLAN missing on one switch in the path | Frames are dropped between switches |
| Number in `dot1Q` differs from the VLAN ID | No communication |

---

## Wireless AP

| Item | Value |
|---|---|
| Device | Access Point0 |
| Switch port | Switch2 Fa0/4, access VLAN 13 |
| VLAN / subnet | 13 / 192.168.13.0/24 |
| Gateway | 192.168.13.1 (Router10 Fa0/0.13) |
| Clients | Smartphone0, Smartphone1 (wireless) |

Wireless clients share VLAN 13, so Router10 treats them as one subnet. That is what makes a per-VLAN ACL possible.

---

## VoIP

Cisco CME on Router11. Phones get an IP and **option 150** (TFTP/CME address) from the router's DHCP pool, then register to `telephony-service`.

<details>
<summary><b>Router11 VoIP config</b></summary>

```text
interface FastEthernet0/0.50
 encapsulation dot1Q 50
 ip address 192.10.51.1 255.255.255.0
!
ip dhcp excluded-address 192.10.51.1 192.10.51.5
ip dhcp pool IP3
 network 192.10.51.0 255.255.255.0
 default-router 192.10.51.1
 option 150 ip 192.10.51.1
!
telephony-service
 max-ephones 5
 max-dn 5
 ip source-address 192.10.51.1 port 2000
 auto assign 1 to 5
!
ephone-dn 1
 number 2002
ephone-dn 2
 number 2003
```
</details>

| Setting | Meaning |
|---|---|
| `max-ephones` / `max-dn` | Maximum phones and directory numbers |
| `ip source-address ... port 2000` | Address and SCCP port phones register to |
| `auto assign 1 to 5` | Assigns DNs to phones automatically |
| `option 150` | Tells phones where the call manager is |
| Switch ports | Access VLAN 50 for phones; uplink to router is a trunk |

---

## ACL

Smartphones (VLAN 13) must not reach Server0 (`100.10.10.2`). An **extended ACL** is applied **inbound** on the VLAN 13 subinterface of Router10, close to the source.

```text
ip access-list extended BLOK-HP
 deny ip 192.168.13.0 0.0.0.255 host 100.10.10.2
 permit ip any any
!
interface FastEthernet0/0.13
 ip access-group BLOK-HP in
```

| Test | Expected |
|---|---|
| Smartphone → `100.10.10.2` | Blocked |
| PC0 / PC1 → `100.10.10.2` | Works |
| `show access-lists` on Router10 | `deny` match counter increases |

`permit ip any any` is required at the end, otherwise the implicit deny blocks all other traffic.

---

## Verification

| Command | Router | Expected |
|---|---|---|
| `show ip bgp summary` | Router0, Router4 | Neighbor shows a prefix count, not `Active` or `Idle` |
| `show ip route bgp` | Router0 | `B` routes to 100.10.10.0/24 and 192.168.x.0/24 |
| `show ip ospf neighbor` | All | State `FULL` |
| `show ip route` | Router10 | `O*E2 0.0.0.0/0 via 10.10.11.2` |
| `show ip interface brief` | All | Interfaces `up/up` |
| `show ephone` | Router11 | Phones `REGISTERED` |
| `show access-lists` | Router10 | Counter increases on `deny` |

Customer PC to ISP PC (`tracert 192.10.15.6`):

```text
1  192.168.12.1   Router10
2  10.10.11.2     Router4
3  200.0.0.1      Router0 (eBGP link)
4  12.10.12.2     Router6
5  192.10.15.6    ISP PC
```

---

## Notes

- Private AS numbers (64512 to 65534) are for labs. Strip them before announcing to the real internet.
- Run `write memory` on every device and save the `.pkt` file.
