---
title: "EIGRP Network Design in the Lab"
date: 2022-01-25T15:50:47-08:00
draft: false
toc: true
images:
  - /blog/images/med-badr.jpg
tags:
  - Cisco
  - EIGRP
---

## Introduction

I just finished the chapter on EIGRP network design in Optimal Routing Design (Cisco Press), I wanted to provide a high level overview of what I learned and incorporated that in the lab. This book really has been great as it provides advanced routing techniques but keeps it in a maintainable format. I hope you enjoy the post and learn something along the way. I would advise you to research on your own and compare the pros and cons. Like anything, it depends.

## The Topology

![EIGRP Topo](/blog/images/eigrp-design.png)

Lets break down the topology above. Everything is in EIGRP AS 12345. I just chose a random number but for the most part, using multiple autonomous systems in EIGRP does not limit the query scope, so we will use just one. Every link is configured as a /31 point to point to save on address space. We will be using a fairly traditional 3 tier design. The DMZ node is advertising a default route throughout the network and has a few loopbacks representing public IP addresses. Example of that below as well as one core nodes routing table before any optimization is performed.

```text
!dmz node redistributing a default route
router eigrp 12345
 network 10.0.0.11 0.0.0.0
 network 10.0.0.13 0.0.0.0
 redistribute static
 eigrp router-id 0.0.0.12
!
ip route 0.0.0.0 0.0.0.0 Null0
```

```text
core3#show ip route eigrp | b Gate
Gateway of last resort is 10.0.0.13 to network 0.0.0.0

D*EX  0.0.0.0/0 [170/281600] via 10.0.0.13, 00:01:21, Ethernet0/3
      10.0.0.0/8 is variably subnetted, 54 subnets, 3 masks
D        10.0.0.0/31 [90/307200] via 10.0.0.4, 00:01:21, Ethernet0/1
                     [90/307200] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.0.0.6/31 [90/307200] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.0.0.10/31 [90/307200] via 10.0.0.13, 00:01:21, Ethernet0/3
                      [90/307200] via 10.0.0.4, 00:01:21, Ethernet0/1
D        10.0.0.14/31 [90/307200] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.0.0.16/31 [90/307200] via 10.0.0.4, 00:01:21, Ethernet0/1
D        10.1.2.0/24 [90/332800] via 10.0.0.4, 00:01:21, Ethernet0/1
                     [90/332800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.3.0/24 [90/332800] via 10.0.0.2, 00:30:01, Ethernet0/0
D        10.1.4.0/24 [90/332800] via 10.0.0.2, 00:30:01, Ethernet0/0
D        10.1.5.0/24 [90/332800] via 10.0.0.2, 00:30:01, Ethernet0/0
D        10.1.30.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.31.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.32.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.33.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.34.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.40.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.41.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.42.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.43.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.44.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.50.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.51.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.52.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.53.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.54.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.1.255.253/32 [90/435200] via 10.0.0.2, 00:30:01, Ethernet0/0
D        10.1.255.254/32 [90/435200] via 10.0.0.4, 00:01:21, Ethernet0/1
D        10.2.3.0/24 [90/332800] via 10.0.0.4, 00:01:21, Ethernet0/1
D        10.2.4.0/24 [90/332800] via 10.0.0.4, 00:01:21, Ethernet0/1
D        10.2.5.0/24 [90/332800] via 10.0.0.4, 00:01:21, Ethernet0/1
D        10.2.30.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.31.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.32.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.33.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.34.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.40.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.41.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.42.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.43.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.44.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.50.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.51.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.52.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.53.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.54.0/24 [90/460800] via 10.0.0.4, 00:01:21, Ethernet0/1
                      [90/460800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        10.2.255.253/32 [90/435200] via 10.0.0.2, 00:30:01, Ethernet0/0
D        10.2.255.254/32 [90/435200] via 10.0.0.4, 00:01:21, Ethernet0/1
      172.16.0.0/16 is variably subnetted, 15 subnets, 3 masks
D        172.16.17.0/31 [90/307200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.17.2/31 [90/307200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.17.8/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.17.16/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.17.24/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.17.32/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.17.253/32 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
                          [90/435200] via 10.0.0.2, 00:01:21, Ethernet0/0
D        172.16.17.254/32 [90/409600] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.18.0/31 [90/332800] via 10.0.0.9, 00:01:21, Ethernet0/2
                        [90/332800] via 10.0.0.2, 00:01:21, Ethernet0/0
D        172.16.18.8/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.18.16/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.18.24/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.18.32/29 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
D        172.16.18.253/32 [90/435200] via 10.0.0.9, 00:01:21, Ethernet0/2
                          [90/435200] via 10.0.0.2, 00:01:21, Ethernet0/0
D        172.16.18.254/32 [90/409600] via 10.0.0.9, 00:01:21, Ethernet0/2
core3#
```

At the moment since no optimizations have been added, if we were to lose any of these links, the network would have to ask every node if they are aware of an alternate path to reach a particular destination. Not very efficient at all. A few places where we can improve are the 10.1.0.0 and 10.2.0.0 networks. Those are all coming from nodes R3, R4, and R5. Think of these as our fake access layer in a campus. The networks in the 172.16.17.0 and 172.16.18.0 networks are coming from a services edge of the network. This could be servers or whatever services you can think of. In this case it is just one node called servers advertising some routes.


## Optimizations in the Core Layer

The main idea behind the core is to push the traffic as fast as possible. Try to limit the extra stuffs that can consume resources within the core. Please note that I said within, do summarize from the core to the aggregation layer. How does this look like in practice? From the aggregation standpoint, all we really need from the core is a default route. One way to do this is to configure a summary on the interfaces towards the aggregates.

```text
!core1
interface Ethernet0/2
 ip summary-address eigrp 12345 0.0.0.0 0.0.0.0
!
interface Ethernet0/3
 ip summary-address eigrp 12345 0.0.0.0 0.0.0.0
```

There’s a catch though. When you create a summary route, EIGRP will add the sweet null0 route… At this point core1 believes it has the better default route over the one the dmz node is sending. For example, lets try and ping the 8.8.8.8 network from svcs1.

```text
svcs1#ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
U.U.U
Success rate is 0 percent (0/5)
svcs1#
```

We have effectively black holed traffic. So how do we get around this? We make that summary route look terrible so it will not be added into the routing table.

```text
!core1-3
router eigrp 12345
 summary-metric 0.0.0.0/0 distance 255
 network 0.0.0.0
```

Please note, you will see the advertise all the things command (network 0.0.0.0) in the configurations. In reality, you should be explicit (network 10.1.2.1 0.0.0.0). Lets try that ping again!

```text
svcs1#ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
svcs1#
```

I’ll add that config and summary metric to all core nodes pointing to their different aggregation layers!

## Optimizations in the Aggregation Layer

A bit similar as before, we can advertise summary routes from the aggregation layer to the core. The catch here is that if one of the aggregation nodes loses access to a route, it will still advertise that summary if a route within the summary is present. The solution here is to connect the aggregation nodes either physically or using a tunnel. I decided to run a direct connection for simplicity. Here is what it looks like to summarize the 10.1.0.0/16 and 10.2.0.0/16 networks from dis1 and dis2. It should be noted, the creation of the null0 route for these summaries is not a big deal, the local network will see more specific routes to reach each destination.

```text
!dis1 and dis2
interface Ethernet1/0
 ip summary-address eigrp 12345 10.1.0.0 255.255.0.0
 ip summary-address eigrp 12345 10.2.0.0 255.255.0.0
```

Similar summaries can be performed from svcs1 and svcs2.

```text
!svcs1 and svcs2
interface Ethernet0/0
 ip summary-address eigrp 12345 172.16.17.0 255.255.255.0
 ip summary-address eigrp 12345 172.16.18.0 255.255.255.0
```

Lets check the access layer and see how a route table is looking.

```text
R3#  show ip route eigrp | b Gate
Gateway of last resort is 10.2.3.2 to network 0.0.0.0

D*EX  0.0.0.0/0 [170/332800] via 10.2.3.2, 00:17:27, Ethernet0/1
      10.0.0.0/8 is variably subnetted, 55 subnets, 3 masks
D        10.0.0.14/31 [90/307200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.0.0.16/31 [90/307200] via 10.2.3.2, 00:57:32, Ethernet0/1
D        10.1.2.0/24 [90/307200] via 10.2.3.2, 00:57:32, Ethernet0/1
                     [90/307200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.4.0/24 [90/307200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.5.0/24 [90/307200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.40.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.41.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.42.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.43.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.44.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.50.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.51.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.52.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.53.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.54.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.255.253/32 [90/409600] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.1.255.254/32 [90/409600] via 10.2.3.2, 00:57:32, Ethernet0/1
D        10.2.4.0/24 [90/307200] via 10.2.3.2, 00:57:32, Ethernet0/1
D        10.2.5.0/24 [90/307200] via 10.2.3.2, 00:57:32, Ethernet0/1
D        10.2.40.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.41.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.42.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.43.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.44.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.50.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.51.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.52.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.53.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.54.0/24 [90/435200] via 10.2.3.2, 00:57:32, Ethernet0/1
                      [90/435200] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.255.253/32 [90/409600] via 10.1.3.1, 00:57:32, Ethernet0/0
D        10.2.255.254/32 [90/409600] via 10.2.3.2, 00:57:32, Ethernet0/1
```

This is a bit overkill for the access layer and we can make improvements here as well. We could add another summary command and metric but lets look at another method. We will limit what is advertised to the access layer using the distribute command coupled with a prefix list. Please see example below.

```text
!dis1 and dis2 pointing towards access
!Similar on svcs1 and svcs2
router eigrp 12345
 distribute-list prefix DEFAULT out Ethernet0/1
 distribute-list prefix DEFAULT out Ethernet0/2
 distribute-list prefix DEFAULT out Ethernet0/3
!
ip prefix-list DEFAULT seq 20 permit 0.0.0.0/0
```

Lets check that routing table again on R3.

```text
R3#show ip route eigrp | b Gate
Gateway of last resort is 10.2.3.2 to network 0.0.0.0

D*EX  0.0.0.0/0 [170/332800] via 10.2.3.2, 00:34:32, Ethernet0/1
R3#
```

## Optimizations in the Access Layer

![Stub](/blog/images/stub.png)

The amount of filtering and summarizing performed already, we have limited the amount of queries EIGRP will send out to search for alternate paths of a route. Another addition that can be made at the access layer is setting the stub flag. Why use the stub feature? Lets say R4 had a loopback1 network of “10.1.48.1/24”. If we were to shut this interface, then we would see a query from dis1 and dis2 towards R3

```text
R3#debug eigrp packets query
    (QUERY)
EIGRP Packet debugging is on
R3#
*Jan 25 20:48:49.461: EIGRP: Received QUERY on Et0/1 - paklen 44 nbr 10.2.3.2
*Jan 25 20:48:49.461:   AS 12345, Flags 0x0:(NULL), Seq 2628/0 interfaceQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
*Jan 25 20:48:49.462: EIGRP: Received QUERY on Et0/0 - paklen 44 nbr 10.1.3.1
*Jan 25 20:48:49.462:   AS 12345, Flags 0x0:(NULL), Seq 976/0 interfaceQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
R3#u all
All possible debugging has been turned off
```

In this example R3 would not have an alternate path to this route and overall we may not want to use these access layer nodes as a transit network to reach other destinations. These devices may have lower bandwidth or overall resources. The stub feature is a neat way to say “do not query me”. Lets test this out!

```text
!Stub setting added to all access layer nodes
R3#show run | s router
router eigrp 12345
 network 0.0.0.0
 eigrp router-id 0.0.0.6
 eigrp stub connected
R3#
R3#debug eigrp packets query
    (QUERY)
EIGRP Packet debugging is on
R3#
```

Okay that’s a bit hard to visualize in writing but test it out and you’ll see that no query is sent to R3.

```text
dis1# show ip eigrp neighbors detail
EIGRP-IPv4 Neighbors for AS(12345)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   10.1.3.3                Et0/1                    14 00:06:35   11   100  0  712
   Version 18.0/2.0, Retrans: 0, Retries: 0, Prefixes: 11
   Topology-ids from peer - 0
   Stub Peer Advertising (CONNECTED ) Routes
   Suppressing queries
```

```text
!dmz gets to be special because it is redistributing a static
router eigrp 12345
 network 10.0.0.11 0.0.0.0
 network 10.0.0.13 0.0.0.0
 redistribute static
 eigrp router-id 0.0.0.12
 eigrp stub static
```

```text
!Almost forgot that core3 routing table!
core3#show ip route eigrp | b Gate
Gateway of last resort is 10.0.0.13 to network 0.0.0.0

D*EX  0.0.0.0/0 [170/281600] via 10.0.0.13, 00:28:48, Ethernet0/3
      10.0.0.0/8 is variably subnetted, 15 subnets, 3 masks
D        10.0.0.0/31 [90/307200] via 10.0.0.4, 02:10:02, Ethernet0/1
                     [90/307200] via 10.0.0.2, 02:10:02, Ethernet0/0
D        10.0.0.6/31 [90/307200] via 10.0.0.2, 02:10:02, Ethernet0/0
D        10.0.0.10/31 [90/307200] via 10.0.0.4, 00:28:52, Ethernet0/1
D        10.0.0.14/31 [90/307200] via 10.0.0.2, 02:10:02, Ethernet0/0
D        10.0.0.16/31 [90/307200] via 10.0.0.4, 02:10:02, Ethernet0/1
D        10.1.0.0/16 [90/435200] via 10.0.0.4, 01:58:15, Ethernet0/1
                     [90/435200] via 10.0.0.2, 01:58:15, Ethernet0/0
D        10.2.0.0/16 [90/435200] via 10.0.0.4, 01:58:15, Ethernet0/1
                     [90/435200] via 10.0.0.2, 01:58:15, Ethernet0/0
      172.16.0.0/24 is subnetted, 2 subnets
D        172.16.17.0 [90/409600] via 10.0.0.9, 01:50:23, Ethernet0/2
D        172.16.18.0 [90/409600] via 10.0.0.9, 01:50:23, Ethernet0/2
core3#
```

## Outro and Links

I originally planned to build this using FRR but it looks like EIGRP is in Alpha stage. I will play with FRR more in the future. Due to this, the topology wont be neatly packaged in a Containerlab file. I will include the configurations below. I hope you all learned something today and thank you for stopping by. We didn’t even go into the new style of configuring EIGRP but I hope you get the point. Named EIGRP just moves some options from the interface level to the af-interface level :). Stay tuned for the next episode of dragon bal…. wait this isn’t that show! In all seriousness, I have something pretty cool planned when we get to OSPF.

- [Featured Image by Med Badr Chenmaoui](https://unsplash.com/photos/ZSPBhokqDMc)
- [EIGRP Configurations zip](/blog/files/eigrp-configs.zip)
