---
title: "Route Redistribution OSPFv3 and EIGRPv6"
date: 2021-05-03
draft: false
toc: true
images:
tags:
  - Aruba
  - Cisco
  - EIGRPv6
  - IPv6
  - OSPFv3
---

## Introduction

Hello and thank you for checking out this post. I was recently working through OSPF and EIGRP for ENARSI studies and something came up on a lab I was building. Redistributing routes between OSPFv3 and EIGRPv6. I am specifically speaking of the IPv6 variant of these protocols.

In the topology you see below. We have three routing devices. One running only EIGRPv6, one running OSPFv3 (Aruba CX), and the redistribution node running both protocols. Both protocols can form neighbors using only link local addressing, so we wont see any global IPv6 addresses on those links.

![OSPF to EIGRP](/blog/images/ospftoeigrp.png)

## OSPFv3

The OSPFv3 node is an Aruba CX switch running …OSPFv3! I have a few loopbacks configured so we can add some networks for testing. All of OSPF will be in area 0 for simplicity. We set all interfaces to passive besides the uplink to REDIS. Configuration snippet is below.

`OSPFv3 Node Configuration`

```text
hostname OSPFv3
!
interface 1/1/1
    no shutdown
    description to REDIS
    ipv6 address link-local fe80::3/64
    ipv6 ospfv3 1 area 0.0.0.0
    no ipv6 ospfv3 passive
    ipv6 ospfv3 network point-to-point
interface loopback 0
    ipv6 address link-local fe80::3/64
    ipv6 address 2001:db8:456:4::1/64
    ipv6 ospfv3 1 area 0.0.0.0
interface loopback 1
    ipv6 address link-local fe80::3/64
    ipv6 address 2001:db8:456:5::1/64
    ipv6 ospfv3 1 area 0.0.0.0
interface loopback 2
    ipv6 address link-local fe80::3/64
    ipv6 address 2001:db8:456:6::1/64
    ipv6 ospfv3 1 area 0.0.0.0
!
router ospfv3 1
    router-id 3.3.3.3
    passive-interface default
    area 0.0.0.0
!
```

## EIGRPv6

In the case of EIGRPv6, we will be using named EIGRP with an address family of IPv6. We add a few loopbacks for testing as well. One difference you’ll see in the configurations, EIGRPv6 is enabled on all interfaces with IPv6 addresses. There is no specific enablement of an interface to run EIGRPv6 or network statements. Configuration snippet is below.

`EIGRPv6 Node Configuration`

```text
hostname EIGRPv6
!
ipv6 unicast-routing
!
interface Loopback0
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:123:1::1/64
!
interface Loopback1
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:123:2::1/64
!
interface Loopback2
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:123:3::1/64
!
interface GigabitEthernet0/0
 description to REDIS
 no ip address
 ipv6 address FE80::1 link-local
!
router eigrp JULIOPDX
 !
 address-family ipv6 unicast autonomous-system 123
  !
  topology base
  exit-af-topology
  eigrp router-id 1.1.1.1
 exit-address-family
```

## REDIS

This node is special, it gets to run two routing protocols and perform redistribution between them! The configuration is very basic indeed. Only two interfaces set for link local addresses. Oh remember when I mentioned EIGRPv6 is enabled for all IPv6 interfaces? I will shutdown the interface towards OSPFv3. Good to reduce all the chatter. Configuration snippet is below.

`REDIS Node Initial Configuration`

```text
hostname REDIS
!
ipv6 unicast-routing
!
interface GigabitEthernet0/0
 description to EIGRPv6
 no ip address
 ipv6 address FE80::2 link-local
!
interface GigabitEthernet0/1
 description to OSPFv3
 no ip address
 ipv6 address FE80::2 link-local
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
!
router eigrp JULIOPDX
 !
 address-family ipv6 unicast autonomous-system 123
  !
  af-interface GigabitEthernet0/1
   shutdown
  exit-af-interface
  !
  topology base
  exit-af-topology
  eigrp router-id 2.2.2.2
 exit-address-family
!
router ospfv3 1
 router-id 2.2.2.2
 !
 address-family ipv6 unicast
  passive-interface default
  no passive-interface GigabitEthernet0/1
 exit-address-family
```

## Neighbors and Routes

`EIGRPv6 Neighbor`

```text
REDIS#show ipv6 eigrp neighbors
EIGRP-IPv6 VR(JULIOPDX) Address-Family Neighbors for AS(123)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                    (sec)         (ms)       Cnt Num
0   Link-local address:     Gi0/0                    11 00:23:40  275  1650  0  11
     FE80::1
REDIS#
```

`EIGRPv6 Routes`

```text
REDIS#show ipv6 route eigrp
D   2001:DB8:123:1::/64 [90/10880]
     via FE80::1, GigabitEthernet0/0
D   2001:DB8:123:2::/64 [90/10880]
     via FE80::1, GigabitEthernet0/0
D   2001:DB8:123:3::/64 [90/10880]
     via FE80::1, GigabitEthernet0/0
REDIS#
```

`OSPFv3 Neighbor`

```text
REDIS#show ipv6 ospf neighbor
        OSPFv3 Router with ID (2.2.2.2) (Process ID 1)
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
3.3.3.3           0   FULL/  -        00:00:35    1360007168      GigabitEthernet0/1
REDIS#
#### Notice, no DR or BDR due to the point to point network
```

`OSPFv3 Routes`

```text
REDIS#show ipv6 route ospf
O   2001:DB8:456:4::1/128 [110/1]
     via FE80::3, GigabitEthernet0/1
O   2001:DB8:456:5::1/128 [110/1]
     via FE80::3, GigabitEthernet0/1
O   2001:DB8:456:6::1/128 [110/1]
     via FE80::3, GigabitEthernet0/1
REDIS#
```

I wont bother sharing the neighbor states of EIGRPv6 or OSPFv3, but below you will see their routes before redistribution.

`Routes on Nodes`

```text
#### EIGRPv6 node first
EIGRPv6#show ipv6 route eigrp
EIGRPv6#
##### Now for OSPFv3 node
OSPFv3# show ipv6 route ospf
No ipv6 routes configured
OSPFv3#
```

## The Redistribution

```text
REDIS#show run | s router
router eigrp JULIOPDX
 !
 address-family ipv6 unicast autonomous-system 123
  !
  topology base
   redistribute ospf 1 metric 1000000 1 255 1 1500
  exit-af-topology
 exit-address-family
router ospfv3 1
 address-family ipv6 unicast
  redistribute eigrp 123
 exit-address-family
```

`New Routes on OSPFv3 and EIGRPv6`

```text
EIGRPv6#show ipv6 route eigrp
EX  2001:DB8:456:4::1/128 [170/15360]
     via FE80::2, GigabitEthernet0/0
EX  2001:DB8:456:5::1/128 [170/15360]
     via FE80::2, GigabitEthernet0/0
EX  2001:DB8:456:6::1/128 [170/15360]
     via FE80::2, GigabitEthernet0/0
EIGRPv6#

OSPFv3# show ipv6 route ospf
2001:db8:123:1::/64, vrf default
        via  fe80::2%1/1/1,  [110/20],  ospf
2001:db8:123:2::/64, vrf default
        via  fe80::2%1/1/1,  [110/20],  ospf
2001:db8:123:3::/64, vrf default
        via  fe80::2%1/1/1,  [110/20],  ospf
OSPFv3#
```

## Simple Ping and Trace

```text
EIGRPv6#ping 2001:db8:456:4::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:456:4::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/18/27 ms
EIGRPv6#traceroute 2001:db8:456:4::1
Type escape sequence to abort.
Tracing the route to 2001:DB8:456:4::1
  1 FE80::2 39 msec 15 msec 16 msec
  2 2001:DB8:456:4::1 13 msec 19 msec 19 msec
EIGRPv6#

OSPFv3# ping6 2001:db8:123:1::1 repetitions 1
PING 2001:db8:123:1::1(2001:db8:123:1::1) 100 data bytes
108 bytes from 2001:db8:123:1::1: icmp_seq=1 ttl=63 time=21.0 ms

--- 2001:db8:123:1::1 ping statistics ---
 1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 21.016/21.016/21.016/0.000 ms
OSPFv3# traceroute6 2001:db8:123:1::1
traceroute to 2001:db8:123:1::1 (2001:db8:123:1::1) from 2001:db8:456:4::1, 30 hops max, 3 sec. timeout, 3 probes, 24 byte packets
 1  fe80::2 (fe80::2)  18.681 ms  13.902 ms  15.768 ms
 2  2001:db8:123:1::1 (2001:db8:123:1::1)  20.081 ms  15.943 ms  19.944 ms
OSPFv3#
```

Thank you for reading this far, I really do appreciate it. I wish you the best and I hope you find something useful or interesting in this post. Cheers!
