---
title: "Aruba Spine Leaf Deployment With OSPFv3 and Link Local Addresses"
date: 2021-03-29
draft: false
toc: true
images:
  - /blog/images/leaf_spine_link_local.png
tags:
  - IPv6
  - Aruba
  - Data Center
  - OSPFv3
authors:
  - Julio Perez
---

![Link Local](/blog/images/leaf_spine_link_local.png)

## Introduction

I was recently on my way to finishing IPv6 Fundamentals by Rick Graziani. I will admit I’m not the fastest reader! In the book Rick mentions the following:

> You could configure router R2’s interfaces with only link-local addresses, no global unicast addresses. This is because R2 has no end user interfaces. RFC 7404, Using Only Link-Local Addressing inside an IPv6 Network, discusses implementing routing protocols using only link-local addresses on infrastructure links.

In my previous post about deploying an Aruba spine leaf, I mentioned the possibility of wasting IP space. In IPv6, we do have an abundance of IP space, but this is still something the operator would have to maintain and work into their workflow or automation. Once I read this, I immediately looked up [RFC 7404](https://datatracker.ietf.org/doc/html/rfc7404) “Using Only Link-Local Addressing inside an IPv6 Network”. The RFC does a really great job of breaking down the pros and cons. In this post we’ll be going over the pros. If want to learn more, feel free to check out the RFC.

Here is a list of a few of the advantages this provides. We’ll be going over most of these in this post.

- Simple address management (automation)
- Lower configuration complexity (automation and reduced errors)
- Smaller routing tables
- Reduced attack surface (less routed links)

## Simple Address Management

If you recall from the previous post, we had to assign the IPv6 equivalent of a point to point address on each spine to leaf connections. This would increase the amount of IP addresses used as well as burden the operator to maintain these configurations. When assigning a link local address on a router, we can essentially assign the same address to each interface. One note as well, I mentioned that Aruba did not support IP unnumbered in the last post. If you are assigning the same address to each interface, we are essentially arriving at the same objective. Below is a sample configuration of a leaf and spine router. Please be aware that we are still configuring a global unicast address (GUA) on a loopback for these devices. This can also be used for device connections (SSH) or handling traffic to NMS.

`Spine01 Configuration`

```text
interface 1/1/1
     no shutdown
     description link to leaf01
     ipv6 address link-local fe80:face:cafe::1/64
     ipv6 ospfv3 1 area 0.0.0.0
     no ipv6 ospfv3 passive
     ipv6 ospfv3 network point-to-point
     ipv6 ospfv3 bfd
 interface 1/1/2
     no shutdown
     description link to leaf02
     ipv6 address link-local fe80:face:cafe::1/64
     ipv6 ospfv3 1 area 0.0.0.0
     no ipv6 ospfv3 passive
     ipv6 ospfv3 network point-to-point
     ipv6 ospfv3 bfd
 interface 1/1/3
     no shutdown
     description link to leaf03
     ipv6 address link-local fe80:face:cafe::1/64
     ipv6 ospfv3 1 area 0.0.0.0
     no ipv6 ospfv3 passive
     ipv6 ospfv3 network point-to-point
     ipv6 ospfv3 bfd
 interface 1/1/4
     no shutdown
     description link to leaf04
     ipv6 address link-local fe80:face:cafe::1/64
     ipv6 ospfv3 1 area 0.0.0.0
     no ipv6 ospfv3 passive
     ipv6 ospfv3 network point-to-point
     ipv6 ospfv3 bfd
 interface loopback 0
     ipv6 address link-local fe80:face:cafe::1/64
     ipv6 address 2001:db8:cafe:ffff::1/128
     ipv6 ospfv3 1 area 0.0.0.0
 !
 router ospfv3 1
     router-id 1.1.1.1
     passive-interface default
     area 0.0.0.0
```

Only the loopback 0 interface on spine01 has a GUA. Essentially, every interface connected to the leaf nodes has the exact same configuration. Easy win for our automation folks!

`Leaf01 Configuration`

```text
lan 1,10
 interface mgmt
     no shutdown
     ip dhcp
 interface 1/1/1
     no shutdown
     description link to spine01
     ipv6 address link-local fe80:beef:cafe::1/64
     ipv6 ospfv3 1 area 0.0.0.0
     no ipv6 ospfv3 passive
     ipv6 ospfv3 network point-to-point
     ipv6 ospfv3 bfd
 interface 1/1/2
     no shutdown
     description link to spine02
     ipv6 address link-local fe80:beef:cafe::1/64
     ipv6 ospfv3 1 area 0.0.0.0
     no ipv6 ospfv3 passive
     ipv6 ospfv3 network point-to-point
     ipv6 ospfv3 bfd
 interface 1/1/6
     no shutdown
     description link to Linux1
     no routing
     vlan access 10
 interface loopback 0
     ipv6 address link-local fe80:beef:cafe::1/64
     ipv6 address 2001:db8:cafe:fd00::1/128
     ipv6 ospfv3 1 area 0.0.0.0
 interface vlan 10
     ipv6 address link-local fe80:beef:cafe::1/64
     ipv6 address 2001:db8:cafe:a::1/64
     no ipv6 nd suppress-ra
     ipv6 ospfv3 1 area 0.0.0.0
 !
 router ospfv3 1
     router-id 10.0.0.1
     passive-interface default
     area 0.0.0.0
```

Difference in leaf nodes are the port VLAN assignments and VLAN interfaces.

## Lower Configuration Complexity

My main background on automation is around Ansible. I will provide some basic YAML variable file as an idea on how the configuration has been simplified. I’ll use spine01 for this example.

`Spine01 variables with GUA addresses`

```yaml
---
hostname: spine01
link_local: fe80:face:cafe::1/64
interfaces:
  1/1/1:
    ipv6_address: 2001:db8:cafe:fe01::a/127
    description: link to leaf01
  1/1/2:
    ipv6_address: 2001:db8:cafe:fe02::a/127
    description: link to leaf02
  1/1/3:
    ipv6_address: 2001:db8:cafe:fe03::a/127
    description: link to leaf03
  1/1/4:
    ipv6_address: 2001:db8:cafe:fe04::a/127
    description: link to leaf04
  loopback0:
    ipv6_address: 2001:db8:cafe:ffff::1/128
```

`Spine01 variables with only link-local`

```yaml
---
hostname: spine01
link_local: fe80:face:cafe::1/64
interfaces:
  1/1/1:
    description: link to leaf01
  1/1/2:
    description: link to leaf02
  1/1/3:
    description: link to leaf03
  1/1/4:
    description: link to leaf04
  loopback0:
    ipv6_address: 2001:db8:cafe:ffff::1/128
```

The example is basic in nature but you can see how mistakes can be reduced and configuration can be simplified.

## Smaller Routing Tables

Since we are using link local addresses, they have a scope that is… local to the link. The routing protocols will not propagate this as reachable networks, this in turn will reduce the total size of our routing tables. This has a great benefit of saving memory and speeding up convergence times. For comparison, check out the routing table on leaf01 when using GUA addresses and only using link local.

`Using GUA /127 addresses`

```text
leaf01# show ipv6 ospfv3 routes | b 2001
  2001:db8:cafe:a::/64 (i) area:0.0.0.0
       directly attached to interface vlan10, cost 100 distance 110
  2001:db8:cafe:14::/64 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 300 distance 110
  2001:db8:cafe:14::/64 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 300 distance 110
  2001:db8:cafe:1e::/64 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 300 distance 110
  2001:db8:cafe:1e::/64 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 300 distance 110
  2001:db8:cafe:28::/64 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 300 distance 110
  2001:db8:cafe:28::/64 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 300 distance 110
  2001:db8:cafe:fd00::2/128 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
  2001:db8:cafe:fd00::2/128 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
  2001:db8:cafe:fd00::3/128 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
  2001:db8:cafe:fd00::3/128 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
  2001:db8:cafe:fd00::4/128 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
  2001:db8:cafe:fd00::4/128 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
  2001:db8:cafe:fe01::a/127 (i) area:0.0.0.0
       directly attached to interface 1/1/1, cost 100 distance 110
  2001:db8:cafe:fe02::a/127 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
  2001:db8:cafe:fe03::a/127 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
  2001:db8:cafe:fe04::a/127 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
  2001:db8:cafe:ff01::a/127 (i) area:0.0.0.0
       directly attached to interface 1/1/2, cost 100 distance 110
  2001:db8:cafe:ff02::a/127 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
  2001:db8:cafe:ff03::a/127 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
  2001:db8:cafe:ff04::a/127 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
  2001:db8:cafe:ffff::1/128 (i) area:0.0.0.0
       via fe80:face:cafe::1 interface 1/1/1, cost 100 distance 110
  2001:db8:cafe:ffff::2/128 (i) area:0.0.0.0
       via fe80:face:cafe::2 interface 1/1/2, cost 100 distance 110
```

`Only link local addresses`

```text
leaf01#   show ipv6 ospfv3 routes | b 2001
 2001:db8:cafe:a::/64 (i) area:0.0.0.0
      directly attached to interface vlan10, cost 100 distance 110
 2001:db8:cafe:14::/64 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 300 distance 110
 2001:db8:cafe:14::/64 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 300 distance 110
 2001:db8:cafe:1e::/64 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 300 distance 110
 2001:db8:cafe:1e::/64 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 300 distance 110
 2001:db8:cafe:28::/64 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 300 distance 110
 2001:db8:cafe:28::/64 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 300 distance 110
 2001:db8:cafe:fd00::2/128 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
 2001:db8:cafe:fd00::2/128 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
 2001:db8:cafe:fd00::3/128 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
 2001:db8:cafe:fd00::3/128 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
 2001:db8:cafe:fd00::4/128 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 200 distance 110
 2001:db8:cafe:fd00::4/128 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 200 distance 110
 2001:db8:cafe:ffff::1/128 (i) area:0.0.0.0
      via fe80:face:cafe::1 interface 1/1/1, cost 100 distance 110
 2001:db8:cafe:ffff::2/128 (i) area:0.0.0.0
      via fe80:face:cafe::2 interface 1/1/2, cost 100 distance 110
```

`Trace from leaf01 and linux04`

```text
leaf01# traceroute6 2001:db8:cafe:28:5200:ff:fe0a:0
 traceroute to 2001:db8:cafe:28:5200:ff:fe0a:0 (2001:db8:cafe:28:5200:ff:fe0a:0) from 2001:db8:cafe:a::1, 30 hops max, 3 sec. timeout, 3 probes, 24 byte packets
  1  2001:db8:cafe:ffff::1 (2001:db8:cafe:ffff::1)  4.712 ms  70.151 ms  24.438 ms
  2  2001:db8:cafe:28::1 (2001:db8:cafe:28::1)  18.437 ms  58.055 ms  17.812 ms
  3  2001:db8:cafe:28:5200:ff:fe0a:0 (2001:db8:cafe:28:5200:ff:fe0a:0)  12.547 ms  53.539 ms  17.951 ms
 leaf01#
```

Last little bit, if you notice the output of the trace, the first hop is the GUA of loopback0 on spine01

## Wrap Up

Thank you for reading this far, I really do appreciate it. If you want to learn more on using link-local addresses between router links, check out [RFC 7404](https://datatracker.ietf.org/doc/html/rfc7404)! Take care and stay safe! Cheers!

- Previous post: [Aruba Spine Leaf with OSPFv3 and IPv6](https://juliopdx.com/2021/03/18/aruba-spine-leaf-with-ospfv3-and-ipv6/)
