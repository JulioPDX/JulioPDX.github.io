---
title: "ExaBGP in the Lab"
date: 2022-02-25T15:06:14-08:00
draft: false
toc: true
mermaid: true
images:
- /blog/images/greg-rakozy.jpg
tags:
  - BGP
  - Docker
  - SR Linux
  - Arista
---

## Intro

I was recently working my way through Optimal Routing design by Cisco Press... Can you tell that I get through books very slowly? I've probably used that sentence in my blog posts three or four times now! Either way, this chapter focused on scaling BGP and dealing with external connections. The section that really got me was external connections. The books goes over a lot, for example, handling inbound traffic, outbound traffic, full routes vs partial vs default. I wanted to see if there was a way to get a sample of the internet routing table in the lab. If you're reading this then the goal was accomplished or I failed miserably and I feel like venting to the world. Either way, if you skipped over this paragraph then you missed my bad humor. Shame on you!

## The Topology

{{< mermaid >}}
graph LR
  1((cmpa1))
  2((cmpa2))
  3((ispa))
  4((ispb))
  5((cloud))
  6((internet))
  1---3
  2---4
  3---5
  4---5
  6---5
{{< /mermaid >}}

Company A(cmpa) has two connections to different ISPs. ISPA is in AS 65001 and ISPB is in AS 65002. Company A also has an iBGP peering between its edge nodes in cmpa1 and cmpa2. In this example I used Arista cEOS nodes as the company routers and Nokia SR Linux at the ISP side.

## Working with ExaBGP

When going down a particular route, I usually always check if something similar has been done before. I really don't like to reinvent the wheel and if someone has created it freely for the world, why not give it a shot! Thankfully I found two really great resources for ExaBGP. One from [Jason Murray](https://jasonmurray.org/posts/2020/exabgp-fulltable/), who made this happen before in GNS3 and a few great posts by [Mat Wood](https://thepacketgeek.com/exabgp/). These posts were incredibly insightful but the step to convert the BGP data from RIPE to an ExaBGP format was time consuming.

### Converting Manual Steps to Dockerfile

I eventually decided on wrapping these steps in a Dockerfile, which could then be used to build a container image. My eventual goal was to use this in a lab with something like Containerlab or Net-sim Tools.

```dockerfile
FROM ubuntu:20.04

# Install dependencies
COPY bgp.cfg .
RUN apt update
RUN apt install python3-pip net-tools wget mrtparse vim nano -y && \
    rm -rf /var/lib/apt/lists/* && apt clean
RUN pip install exabgp==4.2.17

# Download latest table and remove when complete
RUN wget https://data.ris.ripe.net/rrc16/latest-bview.gz
RUN mrt2exabgp -G -P -4 172.16.2.1 latest-bview.gz > fullbgptable.py
RUN rm -rf latest-bview.gz

```

The default configuration file I created for ExaBGP supports two neighbors on AS 65001 and 65002 respectively. Vim and nano are included to allow users to modify this for their deployment. For example you could have ISP nodes on the same AS to simulate multiple external connections to the same ISP.

```text
process announce-routes {
    run python3 fullbgptable.py;
    encoder json;
}

template {
    neighbor AS_65000 {
        router-id 172.16.2.1;
        local-as 65000;
        local-address 172.16.2.1;
    }
}

neighbor 172.16.2.2 {
    inherit AS_65000;
    peer-as 65001;
}

neighbor 172.16.2.3 {
    inherit AS_65000;
    peer-as 65002
}

```

This container is now on [Docker Hub](https://hub.docker.com/repository/docker/juliopdx/exabgp-irt), feel free to pull it down and check it out. After it was on Docker Hub, I had this random idea to automate the updates to the docker image. For example, the BGP view from RIPE is updated a decent amount throughout the day. I think it would be neat to grab the latest update at the end of the day and rebuild the docker image. This could then be tagged with `latest` and the `date` it was created.

### Migrating Docker Build to GitHub Actions

GitHub Actions was fairly new to me, besides some hello world example. I found this neat example from [Chamod Shehanka](https://medium.com/platformer-blog/lets-publish-a-docker-image-to-docker-hub-using-a-github-action-f0b17e5cceb3) that I used as a starting point. I still had to add scheduled updates and syncing the readme/description on Docker Hub with the repository on GitHub. The latter is only available for some pro version of Docker Hub automated builds but luckily there's a GitHub Action to get around this. Example starting workflow file below.

```yaml
name: Docker Image CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:

  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: docker login
      env:
        DOCKER_USER: ${{secrets.DOCKER_USER }}
        DOCKER_PASSWORD: ${{secrets.DOCKER_PASSWORD }}
      run: |
        docker login -u $DOCKER_USER -p $DOCKER_PASSWORD
    - name: Build the Docker image
      run: docker build . --file Dockerfile --tag juliopdx/exabgp-irt:$(date +%s)

    - name: Docker Push
      run: docker push juliopdx/exabgp-irt

```

#### Adding GitHub Actions

Shorty after I discovered two really slick GitHub Actions from some awesome folks to streamline the workflow and arguably make it look a bit cleaner. One being [Docker Build & Push by Sean Smith](https://github.com/marketplace/actions/docker-build-push-action) and the other [sync-readme by Damian Mee](https://github.com/marketplace/actions/docker-hub-readme-description-sync). Both offer a simplicity to the workflow file that I really enjoy. This led me to fairly finalizing this docker workflow.

```yaml
name: Docker Image CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 0 * * *"

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: check out code
        uses: actions/checkout@v2

      - name: build and push docker image
        uses: mr-smithers-excellent/docker-build-push@v5
        with:
          image: juliopdx/exabgp-irt
          registry: docker.io
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASSWORD }}
          tags: latest, $(date +%F)

      - name: sync desc and readme
        uses: meeDamian/sync-readme@v1.0.6
        with:
          user: ${{secrets.DOCKER_USER }}
          pass: ${{ secrets.DOCKER_PASSWORD }}

```

## Incorporating ExaBGP in the Lab

Now that we have the docker image created, we can incorporate it in our lab. For this example I leveraged Containerlab.

```yaml
name: irt
prefix: ""

topology:

  kinds:
    ceos:
      image: ceos:4.27.2F
    srl:
      image: ghcr.io/nokia/srlinux

  nodes:
    ispa:
      kind: srl
      startup-config: conf/ispa.json
      mgmt_ipv4: 172.20.20.11
    ispb:
      kind: srl
      startup-config: conf/ispb.json
      mgmt_ipv4: 172.20.20.12
    internet: # ifconfig eth1 172.16.2.1 netmask 255.255.255.0 up
      kind: linux
      image: juliopdx/exabgp-irt
      mgmt_ipv4: 172.20.20.13
    cmpa1:
      kind: ceos
      startup-config: conf/cmpa1.conf
      mgmt_ipv4: 172.20.20.14
    cmpa2:
      kind: ceos
      startup-config: conf/cmpa2.conf
      mgmt_ipv4: 172.20.20.15
    cloud:
      kind: ceos
      mgmt_ipv4: 172.20.20.16

  links:
    - endpoints: ["cloud:eth1", "internet:eth1"]
    - endpoints: ["cloud:eth2", "ispa:e1-1"]
    - endpoints: ["cloud:eth3", "ispb:e1-1"]
    - endpoints: ["ispa:e1-2", "cmpa1:eth1"]
    - endpoints: ["ispb:e1-2", "cmpa2:eth1"]
    - endpoints: ["cmpa1:eth2", "cmpa2:eth2"]

```

For starters I'll configure the internet node(ExaBGP) with an IP address and activate ExaBGP. In this case not advertising routes, just validating peering.

```text
root@internet:/# cat bgp.cfg
#process announce-routes {
#    run python3 fullbgptable.py;
#    encoder json;
#}

template {
    neighbor AS_65000 {
        router-id 172.16.2.1;
        local-as 65000;
        local-address 172.16.2.1;
    }
}

neighbor 172.16.2.2 {
    inherit AS_65000;
    peer-as 65001;
}

neighbor 172.16.2.3 {
    inherit AS_65000;
    peer-as 65002;
}
root@internet:/#
```

```shell
juliopdx@containerlab:~$ docker exec -it internet bash
root@internet:/# ifconfig eth1 172.16.2.1 netmask 255.255.255.0 up
root@internet:/# exabgp bgp.cfg
04:47:01 | 22     | welcome       | Thank you for using ExaBGP
04:47:01 | 22     | version       | 4.2.17
04:47:01 | 22     | interpreter   | 3.8.10 (default, Nov 26 2021, 20:14:08)  [GCC 9.3.0]
04:47:01 | 22     | os            | Linux internet 5.4.0-99-generic #112-Ubuntu SMP Thu Feb 3 13:50:55 UTC 2022 x86_64
04:47:01 | 22     | installation  | /usr/local
04:47:01 | 22     | advice        | environment file missing
04:47:01 | 22     | advice        | generate it using "exabgp --fi > /usr/local/etc/exabgp/exabgp.env"
04:47:01 | 22     | cli           | could not find the named pipes (exabgp.in and exabgp.out) required for the cli
04:47:01 | 22     | cli           | we scanned the following folders (the number is your PID):
04:47:01 | 22     | cli control   |  - /run/exabgp/
04:47:01 | 22     | cli control   |  - /run/0/
04:47:01 | 22     | cli control   |  - /run/
04:47:01 | 22     | cli control   |  - /var/run/exabgp/
04:47:01 | 22     | cli control   |  - /var/run/0/
04:47:01 | 22     | cli control   |  - /var/run/
04:47:01 | 22     | cli control   |  - /usr/local/run/exabgp/
04:47:01 | 22     | cli control   |  - /usr/local/run/0/
04:47:01 | 22     | cli control   |  - /usr/local/run/
04:47:01 | 22     | cli control   |  - /usr/local/var/run/exabgp/
04:47:01 | 22     | cli control   |  - /usr/local/var/run/0/
04:47:01 | 22     | cli control   |  - /usr/local/var/run/
04:47:01 | 22     | cli control   | please make them in one of the folder with the following commands:
04:47:01 | 22     | cli control   | > mkfifo //run/exabgp.{in,out}
04:47:01 | 22     | cli control   | > chmod 600 //run/exabgp.{in,out}
04:47:01 | 22     | configuration | performing reload of exabgp 4.2.17
04:47:01 | 22     | reactor       | loaded new configuration successfully
04:47:01 | 22     | reactor       | connected to peer-1 with outgoing-1 172.16.2.1-172.16.2.2
04:47:01 | 22     | reactor       | connected to peer-2 with outgoing-1 172.16.2.1-172.16.2.3

```

```text
juliopdx@containerlab:~$ docker exec -it ispa sr_cli
Using configuration file(s): []
Welcome to the srlinux CLI.
Type 'help' (and press <ENTER>) if you need any help using this.
--{ running }--[  ]--
A:ispa# show network-instance default protocols bgp neighbor
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-------------------+---------------------------+-------------------+-------+----------+---------------+---------------+--------------+---------------------------+
|     Net-Inst      |           Peer            |       Group       | Flags | Peer-AS  |     State     |    Uptime     |   AFI/SAFI   |      [Rx/Active/Tx]       |
+===================+===========================+===================+=======+==========+===============+===============+==============+===========================+
| default           | 10.0.0.1                  | cmpa              | S     | 65003    | established   | 0d:0h:4m:51s  | ipv4-unicast | [0/0/2]                   |
| default           | 172.16.2.1                | internet          | S     | 65000    | established   | 0d:0h:3m:21s  | ipv4-unicast | [0/0/2]                   |
+-------------------+---------------------------+-------------------+-------+----------+---------------+---------------+--------------+---------------------------+
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
2 configured neighbors, 2 configured sessions are established,0 disabled peers
0 dynamic peers
--{ running }--[  ]--
A:ispa#
```

```text
juliopdx@containerlab:~$ docker exec -it ispb sr_cli
Using configuration file(s): []
Welcome to the srlinux CLI.
Type 'help' (and press <ENTER>) if you need any help using this.
--{ running }--[  ]--
A:ispb# show network-instance default protocols bgp neighbor
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-------------------+---------------------------+-------------------+-------+----------+---------------+---------------+--------------+---------------------------+
|     Net-Inst      |           Peer            |       Group       | Flags | Peer-AS  |     State     |    Uptime     |   AFI/SAFI   |      [Rx/Active/Tx]       |
+===================+===========================+===================+=======+==========+===============+===============+==============+===========================+
| default           | 10.0.0.2                  | cmpa              | S     | 65003    | established   | 0d:0h:6m:18s  | ipv4-unicast | [0/0/2]                   |
| default           | 172.16.2.1                | internet          | S     | 65000    | established   | 0d:0h:4m:49s  | ipv4-unicast | [0/0/2]                   |
+-------------------+---------------------------+-------------------+-------+----------+---------------+---------------+--------------+---------------------------+
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
2 configured neighbors, 2 configured sessions are established,0 disabled peers
0 dynamic peers
--{ running }--[  ]--
A:ispb#
```

```text
juliopdx@containerlab:~$ docker exec -it cmpa1 Cli
cmpa1>en
cmpa1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.0 4 65001             24        29    0    0 00:10:22 Estab   2      2
  10.0.0.7 4 65003             17        16    0    0 00:10:22 Estab   2      2
cmpa1#
```

### Advertising Many NLRI

At this point all peering has been verified. I will reenable ExaBGP but advertise all the routes this time. This process does take some time to complete and I will explain a few caveats at the end of this post.

```text
root@internet:/# cat bgp.cfg
process announce-routes {
    run python3 fullbgptable.py;
    encoder json;
}

template {
    neighbor AS_65000 {
        router-id 172.16.2.1;
        local-as 65000;
        local-address 172.16.2.1;
    }
}

neighbor 172.16.2.2 {
    inherit AS_65000;
    peer-as 65001;
}

neighbor 172.16.2.3 {
    inherit AS_65000;
    peer-as 65002;
}
root@internet:/# exabgp bgp.cfg
```

I'll save your eyes, the output is a lot! For fun, lets check out some crazy prefix numbers on one ISP node and one Company A node.

```text
A:ispa# show network-instance default protocols bgp summary
------------------------------------------------------------------------------------------------------------------------------------------------
BGP is enabled and up in network-instance "default"
Global AS number  : 65001
BGP identifier    : 172.16.2.2
------------------------------------------------------------------------------------------------------------------------------------------------
  Total paths               : 990
  Received routes           : 74867
  Received and active routes: 74467
  Total UP peers            : 2
  Configured peers          : 2, 0 are disabled
  Dynamic peers             : None
------------------------------------------------------------------------------------------------------------------------------------------------
--{ running }--[  ]--
A:ispa#
```

```text
cmpa1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.0 4 65001           1672       721    0    0 00:21:26 Estab   87670  87670
  10.0.0.7 4 65003           1684      1668    0    0 00:21:26 Estab   86701  86701
cmpa1#
```

That is beautiful, 74K routes on ISPA and 87K routes on Company A node.

## Use Cases

### Reducing Excess Information Client Side

Let me walk through a few examples on how this could be useful in a lab scenario or for general learning. You may have noticed that the Company A node is receiving routes from both ISPA and the iBGP neighbor. Seems like a bit of a waste in resources and memory. We can configure the iBGP neighbors to only send the default route from the ISP. The below configuration is done on both node 1 and 2 of Company A.

```text
ip prefix-list default seq 10 permit 0.0.0.0/0
!
route-map default permit 10
   match ip address prefix-list default
router bgp 65003
   neighbor 10.0.0.7 route-map default out
```

```text
cmpa1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.0 4 65001           8341      7566    0    0 00:31:11 Estab   18898  18898
  10.0.0.7 4 65003           8540      8454    0    0 00:31:11 Estab   1      1
cmpa1#
```

Now we are only receiving the default route from our iBGP neighbor! This iBGP route map also has the benefit of preventing our AS from being a transit AS. Routes learned from one node will not be advertised to the other ISP and possibly severely hindering our edge devices. Maybe we want to filter the routes inbound from the ISPs.

```text
router bgp 65003
   neighbor 10.0.0.0 route-map default in
```

```text
cmpa1#show ip bgp sum
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.0 4 65001          30589      7657    0    0 01:09:18 Estab   257372 1
  10.0.0.7 4 65003           8585      8499    0    0 01:09:17 Estab   1      1
cmpa1#show ip bgp | b Network
          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.0.0.0              0       -          100     0       65001 199524 3257 i
 *        0.0.0.0/0              10.0.0.7              0       -          100     0       65002 199524 3257 i
cmpa1#
```

### Reducing Excess Information Provider Side

Maybe we have a really simple site and we no longer need full tables. Our days of being fancy are over. I'll remove the previous configuration besides the iBGP route-map. Then we will update the ISP to only send the default.

```text
juliopdx@containerlab:~$ docker exec -it ispa sr_cli
Using configuration file(s): []
Welcome to the srlinux CLI.
Type 'help' (and press <ENTER>) if you need any help using this.
--{ running }--[  ]--
A:ispa# enter candidate
--{ candidate shared default }--[  ]--
A:ispa# routing-policy
--{ candidate shared default }--[ routing-policy ]--
A:ispa# set policy default-route default-action reject
A:ispa# set policy default-route statement 10 match prefix-set default
A:ispa# set policy default-route statement 10 action accept
A:ispa# set prefix-set default prefix 0.0.0.0/0 mask-length-range exact
A:ispa# /network-instance default protocols bgp
--{ * candidate shared default }--[ network-instance default protocols bgp ]--
A:ispa# set group cmpa export-policy default-route
A:ispa# commit save
/system:
    Saved current running configuration as initial (startup) configuration '/etc/opt/srlinux/config.json'

All changes have been committed. Leaving candidate mode.
--{ running }--[ network-instance default protocols bgp ]--
A:ispa# show neighbor 10.0.0.1 advertised-routes ipv4
------------------------------------------------------------------------------------------------------------------------------------------------
Peer        : 10.0.0.1, remote AS: 65003, local AS: 65001
Type        : static
Description : None
Group       : cmpa
------------------------------------------------------------------------------------------------------------------------------------------------
Origin codes: i=IGP, e=EGP, ?=incomplete
+---------------------------------------------------------------------------------------------------------------------------------------------+
|          Network                  Next Hop              MED           LocPref                       AsPath                       Origin     |
+=============================================================================================================================================+
| 0.0.0.0/0                    10.0.0.0                    -              100        [65001, 199524, 3257]                            i       |
+---------------------------------------------------------------------------------------------------------------------------------------------+
------------------------------------------------------------------------------------------------------------------------------------------------
1 advertised BGP routes
------------------------------------------------------------------------------------------------------------------------------------------------
--{ running }--[ network-instance default protocols bgp ]--
A:ispa#
```

```text
cmpa1#  show ip bgp sum
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.0 4 65001          35789      7711    0    0 01:30:56 Estab   1      1
  10.0.0.7 4 65003           8615      8526    0    0 01:30:56 Estab   1      1
cmpa1#
```

### Outbound Traffic Steering

#### Partial Routes on Each Node by AS Path Length

Maybe we want to control outbound paths on each Company A node. One node gets routes with AS path length of three or less with a default route. The other will get routes with an AS length of four or more and a default. The iBGP route map will be removed.

```text
# cmpa1
ip prefix-list default seq 10 permit 0.0.0.0/0
!
route-map default permit 5
   match as-path length <= 3
!
route-map default permit 10
   match ip address prefix-list default
!
router bgp 65003
   neighbor 10.0.0.0 route-map default in

# cmpa2
ip prefix-list default seq 10 permit 0.0.0.0/0
!
route-map default permit 5
   match as-path length >= 4 and <= 4000
!
route-map default permit 10
   match ip address prefix-list default
!
router bgp 65003
   neighbor 10.0.0.3 route-map default in
```

I'll share a small sample from active BGP NLRI on each node. Note the AS path lengh on that information.

```text
cmpa1#show ip bgp neighbors 10.0.0.0 routes | b AS Path
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.0.0.0              0       -          100     0       65001 199524 3257 i
 * >      1.0.0.0/24             10.0.0.0              0       -          100     0       65001 52320 13335 i
 * >      1.1.1.0/24             10.0.0.0              0       -          100     0       65001 52320 13335 i
 * >      1.9.0.0/16             10.0.0.0              0       -          100     0       65001 6939 4788 i
 * >      1.12.0.0/20            10.0.0.0              0       -          100     0       65001 52320 132203 i
 * >      1.12.34.0/23           10.0.0.0              0       -          100     0       65001 52320 132203 i
 * >      1.37.0.0/16            10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.0.0/17            10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.27.0/24           10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.32.0/24           10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.35.0/24           10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.36.0/22           10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.47.0/24           10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.86.0/24           10.0.0.0              0       -          100     0       65001 6939 4775 i
 * >      1.37.128.0/17          10.0.0.0              0       -          100     0       65001 6939 4775 i
```

```text
cmpa2#show ip bgp neighbors 10.0.0.3 routes | b AS Path
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.0.0.3              0       -          100     0       65002 199524 3257 i
 * >      1.0.4.0/22             10.0.0.3              0       -          100     0       65002 6939 4826 38803 i
 * >      1.0.4.0/24             10.0.0.3              0       -          100     0       65002 6939 4826 38803 i
 * >      1.0.5.0/24             10.0.0.3              0       -          100     0       65002 6939 4826 38803 i
 * >      1.0.6.0/24             10.0.0.3              0       -          100     0       65002 6939 4826 38803 i
 * >      1.0.7.0/24             10.0.0.3              0       -          100     0       65002 6939 4826 38803 i
 * >      1.0.64.0/18            10.0.0.3              0       -          100     0       65002 52320 1299 2497 7670 18144 i
 * >      1.0.128.0/17           10.0.0.3              0       -          100     0       65002 6939 38040 23969 i
 * >      1.0.128.0/18           10.0.0.3              0       -          100     0       65002 6939 38040 23969 i
 * >      1.0.128.0/19           10.0.0.3              0       -          100     0       65002 6939 38040 23969 i
 * >      1.0.128.0/24           10.0.0.3              0       -          100     0       65002 6939 4651 23969 i
 * >      1.0.129.0/24           10.0.0.3              0       -          100     0       65002 6939 4651 23969 i
 * >      1.0.130.0/23           10.0.0.3              0       -          100     0       65002 6939 38040 23969 i
 * >      1.0.132.0/24           10.0.0.3              0       -          100     0       65002 6939 38040 23969 i
 * >      1.0.133.0/24           10.0.0.3              0       -          100     0       65002 6939 38040 23969 i
```

#### Partial Routes on Each Node by Prefix

Please see the updated configuration on each node and their respective NLRI.

```text
# cmpa1
ip prefix-list default seq 10 permit 0.0.0.0/0
ip prefix-list half seq 10 permit 0.0.0.0/1 le 32
!
route-map default permit 5
   match ip address prefix-list half
!
route-map default permit 10
   match ip address prefix-list default
!
router bgp 65003
   neighbor 10.0.0.0 route-map default in

# cmpa2
ip prefix-list default seq 10 permit 0.0.0.0/0
ip prefix-list half seq 10 permit 128.0.0.0/2 le 32
!
route-map default permit 5
   match ip address prefix-list half
!
route-map default permit 10
   match ip address prefix-list default
!
router bgp 65003
   neighbor 10.0.0.3 route-map default in
```

```text
cmpa1# show ip bgp neighbors 10.0.0.0 routes | b AS Path
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.0.0.0              0       -          100     0       65001 199524 3257 i
 * >      1.0.0.0/24             10.0.0.0              0       -          100     0       65001 52320 13335 i
 * >      1.0.4.0/22             10.0.0.0              0       -          100     0       65001 6939 4826 38803 i
 * >      1.0.4.0/24             10.0.0.0              0       -          100     0       65001 6939 4826 38803 i
 * >      1.0.5.0/24             10.0.0.0              0       -          100     0       65001 6939 4826 38803 i
 * >      1.0.6.0/24             10.0.0.0              0       -          100     0       65001 6939 4826 38803 i
 * >      1.0.7.0/24             10.0.0.0              0       -          100     0       65001 6939 4826 38803 i
 * >      1.0.64.0/18            10.0.0.0              0       -          100     0       65001 52320 1299 2497 7670 18144 i
 * >      1.0.128.0/17           10.0.0.0              0       -          100     0       65001 6939 38040 23969 i
 * >      1.0.128.0/18           10.0.0.0              0       -          100     0       65001 6939 38040 23969 i
```

```text
cmpa2#show ip bgp neighbors 10.0.0.3 routes | b AS Path
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.0.0.3              0       -          100     0       65002 199524 3257 i
 * >      128.0.1.0/24           10.0.0.3              0       -          100     0       65002 52320 9009 i
 * >      128.0.116.0/24         10.0.0.3              0       -          100     0       65002 52320 1299 57878 i
 * >      128.0.118.0/24         10.0.0.3              0       -          100     0       65002 52320 1299 57878 i
 * >      128.1.36.0/24          10.0.0.3              0       -          100     0       65002 6939 4766 i
 * >      128.1.76.0/24          10.0.0.3              0       -          100     0       65002 6939 4766 i
 * >      128.28.0.0/16          10.0.0.3              0       -          100     0       65002 52320 2914 2514 2514 i
 * >      128.45.0.0/16          10.0.0.3              0       -          100     0       65002 52320 1267 24608 i
 * >      128.53.0.0/16          10.0.0.3              0       -          100     0       65002 52320 2914 2514 2514 i
 * >      128.65.96.0/21         10.0.0.3              0       -          100     0       65002 6939 42010 i
```

## Caveats

Now for the issues I had with ExaBGP. I really shouldn't say issues, the things I ran into were mostly user error. I've spent about two days with it and I really like it so far. Like anything I write, this was just a tiny fraction of what ExaBGP can do. If you see some error in the Dockerfile or process in general, please reach out to me. I'd love to update this and get it nicely packaged into something useful. One error I ran into, at some point in the NLRI announcements, ExaBGP will just stop sending or keep a peer session established. It's something I haven't figured out and if someone more familiar with the tool can give me some insight, I'm all ears!

## Outro and Links

Thank you all for reading this far. Really means the world to me and writing this one was a joy. Anytime I get to play with BGP in the lab is a good time. So much to learn with BGP... I tried to include links of posts or documents that helped me throughout this small journey. Please check them out if you have time. Below are some additional links that helped me out a lot!

- [ExaBGP GitHub](https://github.com/Exa-Networks/exabgp)
- [ExaBGP Posts by Mat Wood](https://thepacketgeek.com/exabgp/)
- [ExaBGP fulltable by Jaon Murray](https://jasonmurray.org/posts/2020/exabgp-fulltable/)
- [RRC16 BGP Information](https://data.ris.ripe.net/rrc16/)
- [mrtparse, Convert BGP Information to ExaBGP Format](https://github.com/t2mune/mrtparse/tree/master/examples)
- [Run Actions on a Shedule by Jason Etcovitch](https://jasonet.co/posts/scheduled-actions/)
- [Internet Edge: Traffic Steering Part 1-3 by Peter Welcher](https://netcraftsmen.com/internet-edge-traffic-steering-part-1/)
- [ExaBGP-IRT Docker Hub](https://hub.docker.com/repository/docker/juliopdx/exabgp-irt)
- [Containerlab Clab File Used in Blog](https://github.com/JulioPDX/learning_labs/tree/main/labs/bgp/exabgp)
