---
title: "Multipoint Redistribution and SR Linux"
date: 2022-01-22T15:25:24-08:00
draft: falseS
toc: true
images:
  - /blog/images/pat-krupa.jpg
tags:
  - BGP
  - OSPF
  - Redistribution
  - Containerlab
  - SR Linux
---

![M REDIS](/blog/images/mredis.png)

## Introduction

I’ve been working my way through Optimal Routing Design by Russ White, Don Slice, and Alvaro Retana. It has been a great read so far and I highly recommend it. When I work through a book I usually try and lab up any concept I can, this helps me make it stick as much as possible. Early on in the book the authors mention redistribution and possible issues that can come with doing it at more than one point in the network. So here we are reading this post. I hope you enjoy and possibly learn something along the way.

## Doubling Up the Learning With SR Linux

![Convo With Roman](/blog/images/convo.png)

I’ve been itching to try out SR Linux for some time now. Roman from Nokia kept giving me the friendly nudge to try it out. well, a few days ago it was finally that day! I wont go deep into SR Linux on this post but most all of the examples will use 8 nodes running on my laptop using Containerlab. Nokia was kind enough to make their NOS container publicly available. You can pull down the image with no registrations or sacrificing an email address to spam. Thank you <3.

## Why Redistribute?

Just in case folks reading at home are not familiar with redistribution, I’ll give a high level overview. Lets say you operate a domain that only runs OSPF, some time passes and you need to merge with another domain. Now this domain only runs EIGRP, and for some reason they don’t want to use OSPF. You could point static routes to each other if there is a clean break in IP addresses (unlikely). To make this scale or get around adding a lot of static routes, we can redistribute between OSPF and EIGRP. Essentially letting each domain advertise routes between routing protocols.

## Single Point Redistribution

![Single Point](/blog/images/ospftoeigrp.png)

A while back I wrote a piece on redistributing OSPF to EIGRP, check it out [here](https://juliopdx.com/2021/05/03/route-redistribution-ospfv3-and-eigrpv6/). In single point redistribution, there is little chance of major issues, since there is only one entry and exit point to each domain. Where things get really interesting is when we have multipoint redistribution and that is the focus of this post.

## Multipoint Redistribution

If we were to lose the link at the single point redistribution image above, network connectivity to the other domain would be lost. In this case we can redistribute at multiple points in the network.

### Multipoint Redistribution Topology

![M REDIS](/blog/images/mredis.png)

Above we have 8 SR Linux nodes with the left portion of the network running OSPF as well as the right portion of the network. In the middle everything is running eBGP, each router has an autonomous system of 6500X, where X is the router number. R1 is advertising the network of “10.1.1.0/24” and R8 is advertising network “8.8.8.8/32”. Routers R3, R4, R6, and R7 are redistributing routes between the protocols. I added a simple filter to only advertise the network mentioned in each domain. For example, R3 and R4 will only advertise the network “10.1.1.0/24” into BGP. Example configurations of R3 are below. Please note; R3, R4, R6, and R7 have very similar policies.

```text
A:R3# info routing-policy
    routing-policy {
        prefix-set FILTER {
            prefix 10.1.1.0/24 mask-length-range exact {
            }
        }
        policy OSPFtoBGP {
            statement 10 {
                match {
                    prefix-set FILTER
                }
                action {
                    accept {
                    }
                }
            }
        }
        policy pass-all {
            statement 10 {
                match {
                    protocol bgp
                }
                action {
                    accept {
                    }
                }
            }
        }
    }
```

```text
--{ running }--[ network-instance default protocols ]--
A:R3# info
    bgp {
        autonomous-system 65003
        router-id 0.0.0.3
        group EXTERNAL {
            admin-state enable
            export-policy OSPFtoBGP
            import-policy pass-all
            local-as 65003 {
            }
        }
        neighbor 10.3.5.1 {
            peer-as 65005
            peer-group EXTERNAL
        }
    }
    ospf {
        instance v2 {
            admin-state enable
            version ospf-v2
            router-id 0.0.0.3
            export-policy pass-all
            asbr {
            }
            area 0.0.0.0 {
                interface ethernet-1/1.0 {
                    interface-type point-to-point
                }
            }
        }
    }
```

### Checking Routes at R5

You probably noticed R5 is at the center of this mess, so lets check the routing table from that perspective. Remember R5 is only running BGP.

```text
A:R5# show network-instance default protocols bgp routes ipv4 summary
-------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "default"
-------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale
Origin codes: i=IGP, e=EGP, ?=incomplete
-------------------------------------------------------------------------------------------------------------------------------
+-----+--------------+---------------------+-----+-----+----------------------------------------+
| Sta |   Network    |      Next Hop       | MED | Loc |                Path Val                |
| tus |              |                     |     | Pre |                                        |
|     |              |                     |     |  f  |                                        |
+=====+==============+=====================+=====+=====+========================================+
| u*> | 8.8.8.8/32   | 10.5.6.1            | 16  | 100 | [65006] i                              |
| *   | 8.8.8.8/32   | 10.5.7.1            | 16  | 100 | [65007] i                              |
| u*> | 10.1.1.0/24  | 10.3.5.0            | 32  | 100 | [65003] i                              |
| *   | 10.1.1.0/24  | 10.4.5.0            | 32  | 100 | [65004] i                              |
| u*> | 10.3.5.0/31  | 0.0.0.0             | -   | 100 |  i                                     |
| u*> | 10.4.5.0/31  | 0.0.0.0             | -   | 100 |  i                                     |
| u*> | 10.5.6.0/31  | 0.0.0.0             | -   | 100 |  i                                     |
| u*> | 10.5.7.0/31  | 0.0.0.0             | -   | 100 |  i                                     |
+-----+--------------+---------------------+-----+-----+----------------------------------------+
-------------------------------------------------------------------------------------------------------------------------------
8 received BGP routes: 6 used, 8 valid, 0 stale
6 available destinations: 2 with ECMP multipaths
-------------------------------------------------------------------------------------------------------------------------------
--{ running }--[  ]--
A:R5#
```

So far this looks great, R5 learned about the “8.8.8.8/32” network from R6(10.5.6.1) and R7(10.5.7.1). We can see that R5 also learned about the “10.1.1.0/24” network from R3(10.3.5.0) and R4(10.4.5.0). Lets see if R1 can reach the remote network!

### R1 Reachability to 8.8.8.8/32

```text
A:R1# ping network-instance default -c 4 -I 10.1.1.1 8.8.8.8
Using network instance default
PING 8.8.8.8 (8.8.8.8) from 10.1.1.1 : 56(84) bytes of data.
From 10.2.3.1 icmp_seq=1 Time to live exceeded
From 10.2.3.1 icmp_seq=2 Time to live exceeded
From 10.2.3.1 icmp_seq=3 Time to live exceeded

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 received, +3 errors, 100% packet loss, time 3004ms

--{ running }--[  ]--
A:R1#
```

Hmm, that doesn’t look good. Lets try a trace and maybe we can see whats going on!

```text
A:R1# traceroute network-instance default 8.8.8.8
Using network instance default
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  10.1.2.1 (10.1.2.1)  15.831 ms * *
 2  10.2.3.1 (10.2.3.1)  12.367 ms * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * 10.2.3.1 (10.2.3.1)  15.266 ms *
 9  10.2.3.0 (10.2.3.0)  14.912 ms * *
10  * * *
11  * * *
12  * * *
13  * * *
14  * * *
15  * 10.2.3.0 (10.2.3.0)  12.849 ms *
16  10.2.3.1 (10.2.3.1)  18.093 ms * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * 10.2.3.1 (10.2.3.1)  7.931 ms *
23  10.2.3.0 (10.2.3.0)  5.755 ms * *
24  * * *
25  * * *
26  * * *
27  * * *
28  * * *
29  * 10.2.3.0 (10.2.3.0)  24.179 ms *
30  10.2.3.1 (10.2.3.1)  15.856 ms * *
--{ running }--[  ]--
A:R1#
```

It seems like we reached the max hops, by default 30. At this point, we can most likely come to the conclusion that we have a routing loop. Lets check a few routes and see what they think the best path to reach the “8.8.8.8/32” network is.

```text
# Nothing crazy here, R1 will send traffic towards R2(10.1.2.1)
A:R1# show network-instance default route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : ospfv2
Route Owner   : ospf_mgr
Metric        : 1
Preference    : 150
Best          : true
Last change   : 2022-01-22T18:50:05.915Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.1.2.1 (direct) via [ethernet-1/1.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ running }--[  ]--
A:R1#
```

```text
# Again nothing weird, R2 will go to R3(10.2.3.1) to reach this route
A:R2# show network-instance default route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : ospfv2
Route Owner   : ospf_mgr
Metric        : 1
Preference    : 150
Best          : true
Last change   : 2022-01-22T18:50:05.930Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.2.3.1 (direct) via [ethernet-1/2.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ running }--[  ]--
A:R2#
```

```text
# Thats odd, R3 is sending the route back to R2(10.2.3.0)
A:R3# show network-instance default route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : ospfv2
Route Owner   : ospf_mgr
Metric        : 1
Preference    : 150
Best          : true
Last change   : 2022-01-22T18:50:05.910Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.2.3.0 (direct) via [ethernet-1/1.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ running }--[  ]--
A:R3#
```

This manual validation has shown us that the path R1 will take to reach “8.8.8.8/32” is from R1 -> R2 -> R3 -> R2, a routing loop!

### The problem

When we have multiple points of redistribution there is a chance that loops can be introduced into the network. The reason is that routes advertised into one side of the network can be redistributed right back to our domain, possibly with a better metric. This will then have the effect of a router preferring a local path to a remote network vs using the valid path towards R5.

### The Solutions

There are a few ways to solve this. Some of the more popular ones being filtering, route tagging, and changing administrative distances/preferences. I’ll give an example of tagging later on in this post. Lets see if we can solve this with filtering and changing the preferences first. The goal being, eBGP routes to be preferred over external OSPF routes.

#### Solve With Filtering and Preference

![Allow](/blog/images/mredis-allo.png)

We mentioned before that we are only sending “10.1.1.0/24” into BGP and “8.8.8.8/32” for the other end of the network. We can add a filter to only allow the remote network from BGP into OSPF. Example of configuration below!

```text
A:R3# info routing-policy
    routing-policy {
        prefix-set ALLOW {
            prefix 8.8.8.8/32 mask-length-range exact {
            }
        }
        policy BGPtoOSPF {
            default-action {
                reject {
                }
            }
            statement 10 {
                match {
                    prefix-set ALLOW
                }
                action {
                    accept {
                    }
                }
            }
        }
```

We added a default action reject statement on the policy used to redistribute routes from BGP into OSPF. This will basically deny any routes not matching this policy. Similar statement was added to each router running redistribution. Lets check the connectivity from R3 and R4 standpoint.

```text
--{ + running }--[ network-instance default ]--
A:R3# show route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : ospfv2
Route Owner   : ospf_mgr
Metric        : 1
Preference    : 150
Best          : true
Last change   : 2022-01-22T21:37:23.962Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.2.3.0 (direct) via [ethernet-1/1.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ + running }--[ network-instance default ]--
A:R3#
```

Interesting, R3 is preferring a route to “8.8.8.8/32” with a preference of 150 towards R2. What about R4?

```text
--{ + running }--[ network-instance default ]--
A:R4# show route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : ospfv2
Route Owner   : ospf_mgr
Metric        : 1
Preference    : 150
Best          : true
Last change   : 2022-01-22T21:37:42.270Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.2.4.0 (direct) via [ethernet-1/1.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ + running }--[ network-instance default ]--
A:R4#
```

R4 is also preferring a route towards R2. At this point it seems like a race condition on who wins the route, BGP will see an internal route to reach that network and pick that over an eBGP route. Lets update the preference for external OSPF routes to be worse than 170. Why 170? Lets shutdown OSPF on R3 and see what route R4 uses to reach “8.8.8.8/32”.

```text
--{ + running }--[ network-instance default ]--
A:R4# show route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : bgp
Route Owner   : bgp_mgr
Metric        : 0
Preference    : 170
Best          : true
Last change   : 2022-01-22T21:45:53.701Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.4.5.1 (indirect) resolved by route to 10.4.5.0/31 (local)
  via 10.4.5.0 (direct) via [ethernet-1/2.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ + running }--[ network-instance default ]--
A:R4#
```

We will change the external OSPF preference to be 175 and test if this allows R3 and R4 to prefer the BGP route.

```text
--{ +* candidate shared default }--[ network-instance default protocols ospf ]--
A:R3# info
    instance v2 {
        admin-state disable
        version ospf-v2
        router-id 0.0.0.3
        export-policy BGPtoOSPF
        external-preference 175
        asbr {
        }
        area 0.0.0.0 {
            interface ethernet-1/1.0 {
                interface-type point-to-point
            }
        }
    }
```

We can see below that R3 now prefers the BGP route to reach “8.8.8.8/32”!

```text
--{ + running }--[ network-instance default ]--
A:R3# show route-table ipv4-unicast prefix 8.8.8.8/32 detail
-------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : bgp
Route Owner   : bgp_mgr
Metric        : 0
Preference    : 170
Best          : true
Last change   : 2022-01-22T21:45:52.736Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.3.5.1 (indirect) resolved by route to 10.3.5.0/31 (local)
  via 10.3.5.0 (direct) via [ethernet-1/2.0]
-------------------------------------------------------------------------------------------------------------------------------
Destination   : 8.8.8.8/32
ID            : 0
Route Type    : ospfv2
Route Owner   : ospf_mgr
Metric        : 1
Preference    : 175
Best          : false
Last change   : 2022-01-22T21:57:57.628Z
Resilient hash: false
-------------------------------------------------------------------------------------------------------------------------------
Next hops: 1 entries
10.2.3.0 (direct) via [ethernet-1/1.0]
-------------------------------------------------------------------------------------------------------------------------------
--{ + running }--[ network-instance default ]--
A:R3#
```

The rest of the redistribution nodes have been configured to match these preferences. Lets test connectivity from R1 again!

##### Validating from R1

```text
A:R1# ping network-instance default -c 4 -I 10.1.1.1 8.8.8.8
Using network instance default
PING 8.8.8.8 (8.8.8.8) from 10.1.1.1 : 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=60 time=6.39 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=60 time=16.7 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=60 time=15.4 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=60 time=17.4 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 6.392/13.973/17.422/4.435 ms
--{ running }--[  ]--
A:R1# traceroute network-instance default  8.8.8.8
Using network instance default
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  10.1.2.1 (10.1.2.1)  4.861 ms * *
 2  10.2.3.1 (10.2.3.1)  4.752 ms * *
 3  10.3.5.1 (10.3.5.1)  8.857 ms * *
 4  10.5.6.1 (10.5.6.1)  8.739 ms * *
 5  8.8.8.8 (8.8.8.8)  12.996 ms  12.990 ms  12.971 ms
--{ running }--[  ]--
A:R1#
```

#### Solve with Route Tags

The solution above works but route tags would definitely scale more. A route tag is just a numerical value that can be assigned to routes. The strength comes in when you can query those tags to perform redistribution or policy enforcement. I could not find documentation for tagging routes using SR Linux at the time of this writing, not a knock on them I’m just very new to the NOS. In this case I’ll show some example from Cisco IOS and a high level diagram.

![Route Tags](/blog/images/mredis-tag.png)

Bear with me, I know SR Linux does not run EIGRP :). Here we have the same topology but Cisco nodes are in place. From R3's perspective, when redistributing routes from OSPF to EIGRP, R3 will deny any routes tagged with 444 and set any routes not matching that to 333. In the opposite direction, routes from EIGRP to OSPF with a tag of 444 will be denied, and anything left will be tagged with 333. R4 essentially does the reverse to make sure anything from 333 is not sent to EIGRP or OSPF. Example configurations below.

```text
!R3 Config
router eigrp 34567
 default-metric 1 1 1 1 1
 network 10.3.5.0 0.0.0.0
 redistribute ospf 1 route-map filter-tags
!
router ospf 1
 router-id 0.0.0.3
 redistribute eigrp 34567 subnets route-map filter-tags
 network 10.2.3.1 0.0.0.0 area 0
!
route-map filter-tags deny 10
 match tag 444
!
route-map filter-tags permit 20
 set tag 333
```

```text
!R4 Config
router eigrp 34567
 default-metric 1 1 1 1 1
 network 10.4.5.0 0.0.0.0
 redistribute ospf 1 route-map filter-tags
!
router ospf 1
 router-id 0.0.0.4
 redistribute eigrp 34567 subnets route-map filter-tags
 network 10.2.4.1 0.0.0.0 area 0
!
route-map filter-tags deny 10
 match tag 333
!
route-map filter-tags permit 20
 set tag 444
```

```text
R5#show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "eigrp 34567", distance 170, metric 2560025856
  Tag 444, type external
  Redistributing via eigrp 34567
  Last update from 10.4.5.0 on Ethernet0/1, 00:11:08 ago
  Routing Descriptor Blocks:
  * 10.4.5.0, from 10.4.5.0, 00:11:08 ago, via Ethernet0/1
      Route metric is 2560025856, traffic share count is 1
      Total delay is 1010 microseconds, minimum bandwidth is 1 Kbit
      Reliability 1/255, minimum MTU 1 bytes
      Loading 1/255, Hops 1
      Route tag 444
    10.3.5.0, from 10.3.5.0, 00:11:08 ago, via Ethernet0/0
      Route metric is 2560025856, traffic share count is 1
      Total delay is 1010 microseconds, minimum bandwidth is 1 Kbit
      Reliability 1/255, minimum MTU 1 bytes
      Loading 1/255, Hops 1
      Route tag 333
R5#
```

Here we can see the “10.1.1.0” network from each node with the appropriate tag.

##### Validating from R1 Again

```text
R1#ping 8.8.8.8 source 10.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
Packet sent with a source address of 10.1.1.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
R1#traceroute 8.8.8.8 source 10.1.1.1
Type escape sequence to abort.
Tracing the route to 8.8.8.8
VRF info: (vrf in name/id, vrf out name/id)
  1 10.1.2.1 1 msec 1 msec 1 msec
  2 10.2.3.1 1 msec 0 msec 1 msec
  3 10.3.5.1 2 msec 1 msec 1 msec
  4 10.5.6.1 1 msec 2 msec 1 msec
  5 10.6.8.1 2 msec *  3 msec
R1#
```

## Outro and Links

Thank you all for reading this far, seriously means a lot to me. I hope you found something useful or learned a bit along the way. If I have made any errors in the writing, please let me know! The goal is definitely to be technically accurate. Another option, grab the Github repository where this lab is stored (linked below) and run a pull request! I created a Containerlab repository for this lab and probably future labs I make with Containerlab. Happy labbing and thank you for stopping by!

- [Featured Image by Pat Krupa](https://unsplash.com/photos/Of2rc0KOfVU)
- [GitHub Repository](https://github.com/JulioPDX/learning_labs/tree/main/labs/mpoint_redis)
- [Containerlab](https://containerlab.srlinux.dev/)
