---
title: "RPKI in the Lab"
date: 2023-01-02T14:18:17-08:00
draft: false
toc: true
mermaid: true
images:
  - /blog/images/routinator-ai.jpeg
tags:
  - Arista
  - RPKI
  - BGP
  - Containerlab
---

## Introduction

I was recently finished the last chapters of Optimal Routing Design (great read). One of the chapters covered routing protocol security with protocols such as BGP, EIGRP, and OSPF. This echoed something in my head regarding Resource Public Key Infrastructure (RPKI). I knew the book's printing was around 2003 and figured RPKI wouldn't be in the book. This led me to wonder if I could get RPKI running in the lab, and the following blog post details some of that journey.

I knew very little about RPKI when I started down this road (arguably still know very little). However, I knew it was essential to set up to protect networks from the fate of route hijacks (someone advertising your network space without permission by accident or on purpose).

By default, peering over BGP isn't secure. The workflow is usually Autonomous systems (AS) peering together and start advertising networks. There's a default trust put in place that each peer will advertise the correct routes. Over time, we've developed filtering or route policies to ensure we receive and advertise the proper routes. However, time has shown that these options alone aren't enough to prevent route hijacking or leaks.

## RPKI overview

RPKI provides a way to connect an AS number to IP addresses in its simplest form. The beauty is that this pairing is associated with something we trust. For example, this could be one of the regional internet registries (RIR).

{{< mermaid >}}
graph TD
    subgraph RIR
    AFRINIC
    ARIN
    APNIC
    LACNIC
    RIPENCC
    end
    IANA --> RIR
    AFRINIC --> LIR
    ARIN --> LIR
    APNIC --> LIR
    LACNIC --> LIR
    RIPENCC --> LIR
    LIR --> Customers
{{< /mermaid >}}

The local internet registries (LIR) can create Route Origination Authorizations (ROA). The ROAs is the piece that states what AS can advertise a specific prefix. Once we have valid ROAs, we can determine if a route is valid, invalid, or unknown. You may wonder, "okay, but where do these ROAs get stored?" The ROAs are actually stored in multiple repositories. For example, the RIRs are the trusted source for RPKI information, including ROAs.

If I want to validate IP space given out by APNIC, then I'll subscribe to their repository and trust that the information is accurate and up-to-date. There is so much more involved in RPKI, but I'd like to keep a blog short for once. I'll include some helpful links at the end if you want to learn more.

## Getting RPKI in the lab with Routinator

In this lab, we'll leverage a local Validator of routes. This Validator can pull the latest ROA information from the defined repositories. The Validator we will use is NLnetLabs [Routinator](https://github.com/NLnetLabs/routinator). The Validator will use RPKI to Router (RTR) protocol to signal if a route is valid, invalid, or unknown. This has the benefit of reducing control plane impact on our routers. All the validation and checks are done by the Validator.

{{< mermaid >}}
graph TD
    R1(R1) ---|172.100.100.11|MGMT
    R2(R2) --- |172.100.100.12|MGMT
    RT(Routinator) --- |172.100.100.13|MGMT
{{< /mermaid >}}

### Containerlab to simplify the deployment

NLnetLabs is very kind to include a container image of their Routinator software. Adding this to Containerlab is relatively easy. The example below is our lab file.

```yaml
name: rpki
prefix: ""

mgmt:
  network: statics
  ipv4_subnet: 172.100.100.0/24

topology:
  kinds:
    ceos:
      image: ceos:4.29.0.2F
    linux:
      image: nlnetlabs/routinator
  nodes:
    R1:
      kind: ceos
      mgmt_ipv4: 172.100.100.11
      startup-config: startup/R1.conf
    R2:
      kind: ceos
      mgmt_ipv4: 172.100.100.12
      startup-config: startup/R2.conf
    RPKI:
      kind: linux
      mgmt_ipv4: 172.100.100.13
      ports:
      - 80:8323
  links:
    - endpoints: ["R1:eth1", "R2:eth1"]
```

I kept the topology file as simple and small as possible. We will use two Arista EOS containers to form a BGP relationship and advertise routes. We will then implement RPKI to check the validity of those advertisements. The lab's startup configurations and GitHub repository will be linked at the end of this post.

### Simple BGP config and advertising routes

We will simulate Google and Cloudflare as our two peering AS in our lab. We will advertise a network from each AS and another random network used for testing.

{{< mermaid >}}
flowchart LR
    subgraph AS 15169
    R1(R1)
    end
    subgraph AS 13335
    R2(R2)
    end
    R1 -->|192.168.100.1/32| R2
    R1 -->|8.8.8.0/24| R2
    R2 -->|192.168.100.2/32| R1
    R2 -->|1.1.1.0/24| R1
    linkStyle 0 stroke:green
    linkStyle 1 stroke:green
    linkStyle 2 stroke:green
    linkStyle 3 stroke:green
{{< /mermaid >}}

Below is a simple BGP configuration from R1's perspective.

```text
router bgp 15169
   router-id 192.168.100.1
   neighbor 172.16.0.2 remote-as 13335
   network 8.8.8.0/24
   network 192.168.100.1/32
```

```text
R1#show ip bgp | b Net
          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      1.1.1.0/24             172.16.0.2            0       -          100     0       13335 i
 * >      8.8.8.0/24             -                     -       -          -       0       i
 * >      192.168.100.1/32       -                     -       -          -       0       i
 * >      192.168.100.2/32       172.16.0.2            0       -          100     0       13335 i
R1#
```

Next, we'll add Routinator to our BGP configuration.

### Connecting to Routinator

Setting up our environment with Routinator is relatively simple. Since all nodes in this topology share a management network, we'll use that in our RPKI configuration. Below is the configuration from R1's perspective.

```text
router bgp 15169
   router-id 192.168.100.1
   neighbor 172.16.0.2 remote-as 13335
   network 8.8.8.0/24
   network 192.168.100.1/32
   !
   rpki cache routinator
      host 172.100.100.13 port 3323
      !
      transport tcp
   !
   rpki origin-validation
      ebgp local
```

Once this is implemented, we can check if our router is communicating with our Validator.

```text
R1#show bgp rpki cache
routinator:
Host: 172.100.100.13 port 3323
VRF: default
Refresh interval: 326 seconds
Retry interval: 600 seconds
Expire interval: 7200 seconds
Preference: 5
Protocol version: 1
State: synced
Session ID: 23300
Serial number: 18
Last update sync: never
Last full sync: 0:00:10 ago
Last serial query: never
Last reset query: 0:00:41 ago
Entries: 390484
Connection: Active (0:00:41)
Transport information:
  Protocol: TCP
ROA tables: default


R1#
```

If we check our routes now, we will see two new codes.

- `V`: Valid RPKI routes
- `U`: Unknown RPKI routes

```text
R1#show ip bgp | b Net
          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >    V 1.1.1.0/24             172.16.0.2            0       -          100     0       13335 i
 * >      8.8.8.0/24             -                     -       -          -       0       i
 * >      192.168.100.1/32       -                     -       -          -       0       i
 * >    U 192.168.100.2/32       172.16.0.2            0       -          100     0       13335 i
R1#
```

At this point, we can see that our router is correctly communicating with Routinator and labeling routes as valid or unknown. You may notice that the routes are still in our routing table, even if they are unknown or invalid. In this case, we are simulating the `192.168.100.0` routes as bad. We can correct this by implementing some simple filtering.

### Putting policy and filtering in place

At this point, we can see that our router is correctly communicating with Routinator and labeling routes as valid or unknown. However, you may notice that the routes are still in our routing table, even if they are unknown or invalid. In this case, we are simulating the `192.168.100.0` routes as bad. We can correct this by implementing some simple filtering.

{{< mermaid >}}
flowchart LR
    subgraph AS 135533
    R1(R1)
    end
    subgraph AS 135534
    R2(R2)
    end
    R1 --x|192.168.100.1/32| R2
    R1 -->|8.8.8.0/24| R2
    R2 --x|192.168.100.2/32| R1
    R2 -->|1.1.1.0/24| R1
    linkStyle 0 stroke:red
    linkStyle 1 stroke:green
    linkStyle 2 stroke:red
    linkStyle 3 stroke:green
{{< /mermaid >}}

```text
!
ip as-path access-list 1 permit ^$ any
!
route-map local-only permit 10
   description Only export local routes
   match as-path 1
!
route-map rpki-reject-non-valid deny 10
   description Reject non valid routes
   match origin-as validity invalid
!
route-map rpki-reject-non-valid deny 20
   description Reject not found routes
   match origin-as validity not-found
!
route-map rpki-reject-non-valid permit 100
!
router bgp 15169
   router-id 192.168.100.1
   neighbor 172.16.0.2 remote-as 13335
   neighbor 172.16.0.2 route-map rpki-reject-non-valid in
   neighbor 172.16.0.2 route-map local-only out
```

Below we can see the updates after filtering and compare that with all routes received from our neighbor.

```text
R1#show ip bgp | b Net
          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >    V 1.1.1.0/24             172.16.0.2            0       -          100     0       13335 i
 * >      8.8.8.0/24             -                     -       -          -       0       i
 * >      192.168.100.1/32       -                     -       -          -       0       i
R1#
```

```text
R1#show bgp neighbors 172.16.0.2 received-routes | b AS Path
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >    V 1.1.1.0/24             172.16.0.2            -       -          -       -       13335 i
        U 192.168.100.2/32       172.16.0.2            -       -          -       -       13335 i
R1#
```

If you'd like, you can also check out the Routinator GUI at `http://<Containerlab host>`. Feel free to check out the number of RTR connections or the source for all the repositories.

## Outro and links

That's about it. I hope you learned a little about RPKI and got to play with it in the lab. Please check out the GitHub repository and modify it for other AS and network combos you may want to learn with. Happy new year, and I wish you all the best. Thank you for reading this far. It really means a lot!

- [Featured image by lexica.art](https://lexica.art/)
- [APNIC lab used as inspiration](https://www.sanog.org/resources/sanog33/SANOG33_Tutorials-Routing-Security-Tutorial-RPKI_lab-Tashi-APNIC.pdf)
- [BGP Origin Validation with RPKI by Kenneth Finnegan](https://aristanetworks.force.com/AristaCommunity/s/article/bgp-origin-validation-rpki)
- [Routinator GitHub](https://github.com/NLnetLabs/routinator)
- [RPKI - The required cryptographic upgrade to BGP routing by Martin J Levy](https://blog.cloudflare.com/rpki/)
- [RFC 6480 - An Infrastructure to Support Secure Internet Routing](https://www.rfc-editor.org/rfc/rfc6480#page-11)
- [RPKI on Wikipedia](https://en.wikipedia.org/wiki/Resource_Public_Key_Infrastructure)
- [GitHub repository](https://github.com/JulioPDX/learning_labs/tree/main/labs/bgp/rpki)
