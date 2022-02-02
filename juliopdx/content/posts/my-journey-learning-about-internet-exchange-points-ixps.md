---
title: "My Journey Learning About Internet Exchange Points (IXPs)"
date: 2021-08-29T18:04:42-08:00
draft: false
toc: true
images:
tags:
  - BGP
  - Cisco
  - Cumulus
  - IPv6
  - IXP
  - OSPFv3
---

## Introduction

Hello all and thank you for checking out one of my blog posts. Really means a lot! Have you ever heard IXP mentioned in network engineering speak and thought, what the heck is that? I definitely have, and in my day to day I don’t usually interact with an IXP at all. So here I am a curious individual, wanting to learn the things. I figured, what the heck, now is as good a time as any to deep dive some IXP knowledge. Please note, what I breakdown in this post is just a sliver of what is done at an IXP or even at the internet service provider side.

## What is an IXP?

In the absolute simplest terms… a managed switch. Seriously, that’s pretty much it. I’m only kidding a bit. IXPs allow internet service providers or content providers to have a general space to come together and peer with each other. Each participant will have a router at the IXP site and peer with every participant, route server, or a subset of the participants. Peering will be done using external BGP (eBGP). This can also be a combination of peering with a route server and a few participants. The cost to run an IXP is usually divided between the participants.

## Why do we need them?

Imagine two internet providers with customers of their own in the west coast. The ISPs only peer with each other on the east coast. If customers on the west coast want to talk to each other, that traffic would have to travel from coast to coast just to end up back on the west coast. Not very efficient or economical. The ISPs could peer with each other at the west coast, but what if there’s another ISP or five? This would not scale very well and would become very costly. Below is a simple list of other IXP benefits:

- Save money
- Traffic stays local
- Performance
- Better customer experience

## Our Topology

![Topology](/blog/images/IXP.png)

In the topology above we have a dual site IXP. Why a dual site? In cases where IXPs become very large, the requirement for a second site may arise. I also just wanted to play with a dual site topology for more learnings. Please note the IXP sites are not directly connected. Good for decreasing failure domain.

We have four ISPs, each ISP is advertising IPv4 and IPv6 networks. Every ISP has a three router setup. Routers ISP-X-1 and ISP-X-2 are edge nodes for the ISP and ISP-X-3 is internal to the ISP network (advertising the routes). I’ll mainly focus on ISP-A since all others ISPs have a very similar configurations, sub some IP settings. The IXP site also has a node acting as a route server/router collector. More on that in a bit!

## IXP Network Addressing

|       Site       |            IXP1            |            IXP2             |
| :--------------: | :------------------------: | :-------------------------: |
| IPv4 Peering LAN |        10.0.0.0/23         |         10.0.2.0/23         |
| IPv6 Peering LAN |       2001:db8::/64        |      2001:db8:0:1::/64      |
|       ASN        | 777(should be private ASN) | 777 (should be private ASN) |

I saw some examples mentioning the IXP LAN can be public addressing or private addressing. In my case I chose private addressing for IPv4 and the general documentation prefix for IPv6. The ASN at the IXP should be a private ASN. When I started this build I chose 777 because luck.

## ISP Network Addressing

|    ISP     |         A         |         B         |         C         |         D         |
| :--------: | :---------------: | :---------------: | :---------------: | :---------------: |
| IPv4 Block |    1.1.1.0/24     |    2.2.2.0/24     |    3.3.3.0/24     |    4.4.4.0/24     |
| IPv6 Block | 2001:db8:111::/48 | 2001:db8:222::/48 | 2001:db8:333::/48 | 2001:db8:444::/48 |
|    ASN     |        111        |        222        |        333        |        444        |

Lets just pretend in our fun world that these fictional ISPs own these public addresses. The ISP-X-3 router at each ISP site will be advertising the networks shown above.

## ISP Topology

![ISP Edge](/blog/images/ISP-Edge.png)

Each ISP has two edge routers and one router internal to the ISP autonomous system. Each ISP is running OSPFv3 to redistribute internal routes and addresses on loopback interfaces for BGP peering. The internal router, in this case ISP-A-3, acts as a route reflector. A-1 and A-2 do not directly peer with each other using BGP. They are neighbors using OSPFv3 IPv4 and IPv6 address families.

## OSPFv3 Configuration

`ISP-A-3 OSPFv3 Configuration`

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.0
 ipv6 address 2001:DB8:111::1/48
!
interface Loopback1
 ip address 192.168.1.3 255.255.255.255
 ipv6 address FE80::3 link-local
 ipv6 address 2001:DB8:1::3/128
 ospfv3 1 ipv6 area 0
 ospfv3 1 ipv4 area 0
!
interface Ethernet0/0
 description To ISP-A-1
 ip address 192.168.0.3 255.255.255.254
 ipv6 address FE80::3 link-local
 ospfv3 1 ipv4 area 0
 ospfv3 1 ipv6 area 0
 bfd interval 300 min_rx 300 multiplier 3
!
interface Ethernet0/1
 description To ISP-A-2
 ip address 192.168.0.5 255.255.255.254
 ipv6 address FE80::3 link-local
 ospfv3 1 ipv4 area 0
 ospfv3 1 ipv6 area 0
 bfd interval 300 min_rx 300 multiplier 3
!

router ospfv3 1
 router-id 0.0.0.3
 bfd all-interfaces
 !
 address-family ipv4 unicast
  passive-interface Loopback0
  passive-interface Loopback1
 exit-address-family
 !
 address-family ipv6 unicast
  passive-interface Loopback0
  passive-interface Loopback1
 exit-address-family
!
```

IPv4 addressing is the same across all ISPs in the topology as well as the link local addressing. We enable OSPFv3 under both address families and set loopbacks to passive in OSPFv3. I enabled BFD on pretty much any link I could to speed up failure detection, especially with BGP peers.

## iBGP Configuration

`ISP-A-3 iBGP Configuration`

```text
router bgp 111
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor iBGP peer-group
 neighbor iBGP remote-as 111
 neighbor iBGP update-source Loopback1
 neighbor iBGP fall-over bfd
 neighbor iBGPv6 peer-group
 neighbor iBGPv6 remote-as 111
 neighbor iBGPv6 update-source Loopback1
 neighbor iBGPv6 fall-over bfd
 neighbor 2001:DB8:1::1 peer-group iBGPv6
 neighbor 2001:DB8:1::2 peer-group iBGPv6
 neighbor 192.168.1.1 peer-group iBGP
 neighbor 192.168.1.2 peer-group iBGP
 !
 address-family ipv4
  network 1.1.1.0 mask 255.255.255.0
  neighbor iBGP route-reflector-client
  neighbor 192.168.1.1 activate
  neighbor 192.168.1.2 activate
 exit-address-family
 !
 address-family ipv6
  network 2001:DB8:111::/48
  neighbor iBGPv6 route-reflector-client
  neighbor 2001:DB8:1::1 activate
  neighbor 2001:DB8:1::2 activate
 exit-address-family
```

This configuration looks a bit long but lets break it down. We disable the default BGP for IPv4 setup. We will be using both address families for neighbor relationships. There is a peer group for IPv4 neighbors and IPv6 neighbors to cut down on commands. Especially if more neighbors were involved. After that we will advertise the routes we need and set the peers in the peer groups to be route reflector clients.

`ISP-A-1 iBGP Configuration`

```text
router bgp 111
 neighbor 2001:DB8:1::3 remote-as 111
 neighbor 2001:DB8:1::3 update-source Loopback0
 neighbor 2001:DB8:1::3 fall-over bfd
 neighbor 192.168.1.3 remote-as 111
 neighbor 192.168.1.3 update-source Loopback0
 neighbor 192.168.1.3 fall-over bfd
 !
 address-family ipv4
  neighbor 192.168.1.3 activate
  neighbor 192.168.1.3 next-hop-self
  neighbor 192.168.1.3 route-map iBGP_IPv4_IN in
 exit-address-family
 !
 address-family ipv6
  neighbor 2001:DB8:1::3 activate
  neighbor 2001:DB8:1::3 next-hop-self
  neighbor 2001:DB8:1::3 route-map iBGP_IPv6_IN in
 exit-address-family
```

I’ll spare you from the configuration on ISP-A-2, it is so close to being a copy of ISP-A-1. Not much new here, just enabling the neighbor relationship with ISP-A-3. Something new is the route-map being applied inbound. One of the resources I used mentioned the ISP border routers at the IXP should, for the most part, not carry the full internet routing table. Lets say for example, ISPA wanted to advertise only networks 1.1.1.0/24 and 2001:db8:111::/48 at this IXP. I accomplished this by filtering all incoming updates from our iBGP neighbor to only accept those prefixes. See prefix lists and route maps below.

`Inbound Prefix Lists and Route Maps`

```text
!
ip prefix-list INBOUND seq 5 permit 1.1.1.0/24
!
ipv6 prefix-list INBOUND_v6 seq 5 permit 2001:DB8:111::/48
!
route-map iBGP_IPv4_IN permit 10
 match ip address prefix-list INBOUND
!
route-map iBGP_IPv6_IN permit 10
 match ipv6 address prefix-list INBOUND_v6
!
```

## eBGP Configuration

`ISP-A-1 eBGP Configuration`

```text
router bgp 111
 bgp router-id 10.0.0.21
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor IXP peer-group
 neighbor IXP fall-over bfd
 neighbor IXPv6 peer-group
 neighbor IXPv6 fall-over bfd
 neighbor 10.0.0.1 remote-as 777
 neighbor 10.0.0.1 peer-group IXP
 neighbor 10.0.0.22 remote-as 222
 neighbor 10.0.0.22 peer-group IXP
 neighbor 10.0.0.23 remote-as 333
 neighbor 10.0.0.23 peer-group IXP
 neighbor 10.0.0.24 remote-as 444
 neighbor 10.0.0.24 peer-group IXP
 neighbor 2001:DB8::1 remote-as 777
 neighbor 2001:DB8::1 peer-group IXPv6
 neighbor 2001:DB8::222:1 remote-as 222
 neighbor 2001:DB8::222:1 peer-group IXPv6
 neighbor 2001:DB8::333:1 remote-as 333
 neighbor 2001:DB8::333:1 peer-group IXPv6
 neighbor 2001:DB8::444:1 remote-as 444
 neighbor 2001:DB8::444:1 peer-group IXPv6
 !
 address-family ipv4
  network 1.1.1.0 mask 255.255.255.0
  neighbor IXP send-community both
  neighbor IXP soft-reconfiguration inbound
  neighbor IXP route-map IXP_IPv4_OUT out
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.22 activate
  neighbor 10.0.0.23 activate
  neighbor 10.0.0.24 activate
 exit-address-family
 !
 address-family ipv6
  network 2001:DB8:111::/48
  neighbor IXPv6 send-community both
  neighbor IXPv6 soft-reconfiguration inbound
  neighbor IXPv6 route-map IXP_IPv6_OUT out
  neighbor 2001:DB8::1 activate
  neighbor 2001:DB8::222:1 activate
  neighbor 2001:DB8::333:1 activate
  neighbor 2001:DB8::444:1 activate
 exit-address-family
!
```

I'll admit the eBGP configuration is a bit more busy. Here we have peer groups for both IPv4 (IXP) and IPv6 (IXPv6). Sending community attributes for potential traffic engineering or policies. Something that I should mention for more official implementation, authentication between BGP peers would be great as well as filtering per neighbor. For example, if we were to peer with 10.0.0.22 and expect to receive the 2.2.2.0/24 network. We could add a filter to only accept that network inbound from 10.0.0.22. I created a simple filter to only filter the routes we are sending to our eBGP neighbors. Example is below.

`Outbound Prefix Lists and Route Maps`

```text
!
ip prefix-list OUTBOUND seq 5 permit 1.1.1.0/24
!
ipv6 prefix-list OUTBOUND_v6 seq 5 permit 2001:DB8:111::/48
route-map IXP_IPv4_OUT permit 10
 match ip address prefix-list OUTBOUND
!
route-map IXP_IPv6_OUT permit 10
 match ipv6 address prefix-list OUTBOUND_v6
!
```

## IXP Switching

I did not go deep in the reading for IXP hardware. My best thought would be a pretty decent switch with speeds up to 400G at this point. I’m sure there are a lot of factors that go into these decisions. Check out the IXP wish list from euro-ix linked at the end.

I’m going to turn our attention to the actual switch configuration. This is again only a sliver of what should be configured on the switch. Below is what I came up with in this lab scenario.

`IXP Switchport Configuration`

```text
!
interface Ethernet0/0
 description To ISP-A-1
 switchport access vlan 1234
 switchport mode access
 switchport port-security
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky aabb.cc00.0310
 duplex auto
 no cdp enable
 spanning-tree portfast
 spanning-tree bpdufilter enable
!
interface Ethernet1/0
 description To RS1
 switchport access vlan 1234
 switchport mode access
 switchport port-security
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5000.000b.0001
 duplex auto
 spanning-tree portfast
 ip dhcp snooping trust
!
```

I included just the ports to ISP-A-1 and RS1 for brevity and save your eyes a bit. VLAN 1234 will be used for peering. Portfast is enabled to bring up the interfaces quicker. Port security and MAC address sticky is used to prevent a rogue device from being connected and trying to join our peering network. In our case the single MAC will be remembered even on a switch reboot. I set the mode to restrict in this lab, this should allow the port to stay up and packets from rogue MAC addresses will be dropped. Alerts can then be sent to a network management system. I disabled CDP just in case some rogue device gets access, they at least wont know what network the neighbor device is on. BPDU filter is on just in case some rogue switch is connected and prevent things from going horribly wrong. All other interfaces not in use are shut down.

## IXP Services

Apart from giving ISPs or content providers the ability to peer with each other. IXPs can offer several additional services:

- Country Code Top Level Domain – Ability to host country’s top level DNS
- Root Servers – Used to reduce latency of DNS resolutions
- Route Collector – Ability to view the routing information at the IXP
- Looking Glass – Public or members only view of the Route Collector routing information
- NTP – Possibility to host a stratum 1 time source

## Route Collector

The services mentioned above are only a handful of what an IXP can provide. I’m going to focus on one in particular, the Route Collector. An IXP can have a policy for each participant that they must peer with a local route collector. The collector has a simple function, receive all the routes, and distribute none. You may have noticed two nodes in the topology named RS1 and RS2. The name may be misleading as it can stand for route server, but for our purposes just think of them as a route collector. If you are wondering, route servers can make BGP configuration a bit more simple for ISPs at an IXP since they only have to peer with the route server vs each individual participant. This has a possible negative of the ISP losing a bit of policy control with BGP.

Our route collectors are Cumulus Linux VMs running a neat BGP setup. I’m not entirely sure on the correct way to setup the route collector but this made sense to me. Since the route collector cannot forward any routes that are learned, we need to create a policy. The most simple thing that came to mind was to create a general deny all route map and apply it to our peer group. Check it out below. I’ll focus on RS1 as RS2 has a very very similar config. So much so that I basically ran “net show configuration commands”, copied the output, updated about two addresses, and then pasted them over to RS2.

`RS1 BGP Configuration`

```text
router bgp 777
  no bgp default ipv4-unicast
  neighbor IXP peer-group
  neighbor IXP remote-as external
  neighbor IXP bfd 3 300 300
  neighbor IXPv6 peer-group
  neighbor IXPv6 remote-as external
  neighbor IXPv6 bfd 3 300 300
  bgp listen limit 90
  bgp listen range 10.0.0.0/23 peer-group IXP
  bgp listen range 2001:db8::/64 peer-group IXPv6

  address-family ipv4 unicast
    neighbor IXP activate
    neighbor IXP soft-reconfiguration inbound
    neighbor IXP route-map RS_DENY out
    neighbor IXP send-community both

  address-family ipv6 unicast
    neighbor IXPv6 activate
    neighbor IXPv6 soft-reconfiguration inbound
    neighbor IXPv6 route-map RS_DENY out
    neighbor IXPv6 send-community both

route-map RS_DENY deny 1
```

Okay so the config was small enough I just included the whole thing. At the bottom you can see my generic deny route map. Check out those bgp listen commands, I think its the coolest thing. In my little IXP world, I didn’t want to keep manually adding all the BGP peers to the route collector. So the route collector is listening for any peer requests that come from our peering LAN. Probably not the most secure, I would recommend reading up on the best way to set that up.

## Validate

Lets check some neighbors and routes!

`ISP-A-3 OSPFv3 Neighbors`

```text
ISP-A-3#show ospfv3 neighbor

          OSPFv3 1 address-family ipv4 (router-id 0.0.0.3)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
0.0.0.2           1   FULL/BDR        00:00:35    5               Ethernet0/1
0.0.0.1           1   FULL/BDR        00:00:37    5               Ethernet0/0

          OSPFv3 1 address-family ipv6 (router-id 0.0.0.3)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
0.0.0.2           1   FULL/BDR        00:00:34    5               Ethernet0/1
0.0.0.1           1   FULL/BDR        00:00:33    5               Ethernet0/0
```

`ISP-A-3 BGP Neighbors`

```text
ISP-A-3#show bgp ipv4 unicast summary | b Nei
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.1     4          111    1439    1429       14    0    0 21:30:04        3
192.168.1.2     4          111    1370    1373       14    0    0 20:36:57        3
ISP-A-3#show bgp ipv6 unicast summary | b Nei
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
2001:DB8:1::1   4          111    1474    1424       43    0    0 21:11:37        3
2001:DB8:1::2   4          111    1486    1440       43    0    0 21:11:36        3
ISP-A-3#
```

`ISP-A-3 BGP Networks`

```text
ISP-A-3#show bgp ipv4 unicast | b Net
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                  0         32768 i
 * i 2.2.2.0/24       192.168.1.2              0    100      0 222 i
 *>i                  192.168.1.1              0    100      0 222 i
 * i 3.3.3.0/24       192.168.1.2              0    100      0 333 i
 *>i                  192.168.1.1              0    100      0 333 i
 * i 4.4.4.0/24       192.168.1.2              0    100      0 444 i
 *>i                  192.168.1.1              0    100      0 444 i
ISP-A-3#show bgp ipv6 unicast | b Net
     Network          Next Hop            Metric LocPrf Weight Path
 *>  2001:DB8:111::/48
                       ::                       0         32768 i
 * i 2001:DB8:222::/48
                       2001:DB8:1::2            0    100      0 222 i
 *>i                  2001:DB8:1::1            0    100      0 222 i
 * i 2001:DB8:333::/48
                       2001:DB8:1::2            0    100      0 333 i
 *>i                  2001:DB8:1::1            0    100      0 333 i
 * i 2001:DB8:444::/48
                       2001:DB8:1::2            0    100      0 444 i
 *>i                  2001:DB8:1::1            0    100      0 444 i
ISP-A-3#
```

`RS1 BGP Neighbors`

```text
cumulus@RS1:mgmt:~$ net show bgp summary
show bgp ipv4 unicast summary
=============================
BGP router identifier 10.0.0.1, local AS number 777 vrf-id 0
BGP table version 72
RIB entries 7, using 1344 bytes of memory
Peers 4, using 85 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
*10.0.0.21      4        111     27760     28419        0    0    0 23:41:01            1        0
*10.0.0.22      4        222     63678     65191        0    0    0 2d06h19m            1        0
*10.0.0.23      4        333     63678     65191        0    0    0 2d06h19m            1        0
*10.0.0.24      4        444     28302     28971        0    0    0 1d00h08m            1        0

Total number of neighbors 4
* - dynamic neighbor
4 dynamic neighbor(s), limit 90


show bgp ipv6 unicast summary
=============================
BGP router identifier 10.0.0.1, local AS number 777 vrf-id 0
BGP table version 49
RIB entries 7, using 1344 bytes of memory
Peers 4, using 85 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor         V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
*2001:db8::111:1 4        111     27760     28417        0    0    0 23:40:55            1        0
*2001:db8::222:1 4        222     63716     65191        0    0    0 2d06h19m            1        0
*2001:db8::333:1 4        333     63715     65191        0    0    0 2d06h19m            1        0
*2001:db8::444:1 4        444     63714     65191        0    0    0 2d06h19m            1        0

Total number of neighbors 4
* - dynamic neighbor
4 dynamic neighbor(s), limit 90


show bgp l2vpn evpn summary
===========================
% No BGP neighbors found
cumulus@cumulus:mgmt:~$
```

`RS1 BGP Routes`

```text
cumulus@cumulus:mgmt:~$  net show bgp
show bgp ipv4 unicast
=====================
BGP table version is 72, local router ID is 10.0.0.1, vrf id 0
Default local pref 100, local AS 777
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.0/24       10.0.0.21                              0 111 i
*> 2.2.2.0/24       10.0.0.22                              0 222 i
*> 3.3.3.0/24       10.0.0.23                              0 333 i
*> 4.4.4.0/24       10.0.0.24                              0 444 i

Displayed  4 routes and 4 total paths


show bgp ipv6 unicast
=====================
BGP table version is 49, local router ID is 10.0.0.1, vrf id 0
Default local pref 100, local AS 777
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 2001:db8:111::/48
                    fe80::a8bb:ccff:fe00:310
                                                           0 111 i
*> 2001:db8:222::/48
                    fe80::a8bb:ccff:fe00:510
                                                           0 222 i
*> 2001:db8:333::/48
                    fe80::a8bb:ccff:fe00:710
                                                           0 333 i
*> 2001:db8:444::/48
                    fe80::a8bb:ccff:fe00:810
                                                           0 444 i

Displayed  4 routes and 4 total paths
cumulus@cumulus:mgmt:~$
```

## Outro and Links

Thank you again for checking out this post, I really appreciate it. Please send over any tips or just ping me if something I mentioned is way out into left field. If you are curious about IXPs, please check out the links below! I will include all configurations for the devices in the topology on my GitHub (linked below). Thank you to all the IXP operators around the world!

- [PACNOG – Internet Exchange Point Design](https://www.pacnog.org/pacnog6/IXP/IXP-design.pdf)
- [Network Startup Resource Center – IXP Design and Implementation](https://learn.nsrc.org/bgp/ixp_history)
- [EURO-IX – Internet Exchange Point Wishlist](https://www.euro-ix.net/media/filer_public/0a/5b/0a5b4a4e-e032-41f8-b0f7-43c1375c5442/ixp-wishlist.pdf)
- [Configuration Files](https://github.com/JulioPDX/IXP_Learnings)
