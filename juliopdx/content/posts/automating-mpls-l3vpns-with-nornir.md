---
title: "Automating MPLS L3VPNs With Nornir"
date: 2021-08-14T10:38:25-08:00
draft: false
toc: true
images:
- /blog/images/auto_mpls.png
tags:
  - Cisco
  - L3VPN
  - MPLS
  - Network Automation
  - Python
---

![Topology](/blog/images/auto_mpls.png)

## Introduction

Hello all and thank you for checking out another one of my blog posts. It really means a lot. I recently completed the Cisco ENARSI exam after a few months of studying. It was honestly pretty difficult for me but the pass was worth it! I may have to write a blog post on studying for and passing that exam. In this post I will break down how I automated some parts of an MPLS L3VPN deployment. Configuring the provider edge devices to allow connectivity between customer edge routers.

In serious news, that was the intro you see in those recipe websites where you just want to see the ingredients and the cooking directions but there’s a whole life story on the dish or how it came about…. well this blog post is similar! Usually when studying for an exam I tend to get tunnel vision and focus on the task at hand. That also had me dropping the love for automation a bit. One of the main points in ENARSI is learning about MPLS and more specifically MPLS L3VPNs. I had such a joy learning more about this topic and running through multiple labs to get used to the technology. I wont go too deep into MPLS in this post since I’m assuming the reader is trying to automate that exact task or is familiar with it already.

## Topology Brief

As you can see by the featured image in this post, we have four provider edge (PE) routers and four customer edge (CE) routes. Each PE device connects to one CE device, and all PE routers connect to the provider (P) router, also acting as our BGP route reflector. You might be thinking, why the route reflector? You could run iBGP sessions between all PE devices but down the road that can lead to a lot of overhead. In this case all PE devices only have one internal BGP relationship between the P router and one external BGP peering with the CE routers. This also makes the iBGP configuration on the PE routers very cookie cutter. In this case we are pretending not to have control of the CE devices, so those will already be preconfigured. I will include the configurations on the nodes so you can see what is all preconfigured.

## Ce Routers Configuration Example

```shell
interface Loopback0
 ip address 172.16.1.1 255.255.255.0
!
interface Ethernet0/0
 ip address 10.0.11.2 255.255.255.0
!

router bgp 789
 bgp log-neighbor-changes
 network 172.16.1.0 mask 255.255.255.0
 neighbor 10.0.11.1 remote-as 12345
 neighbor 10.0.11.1 allowas-in
```

Every CE router looks very similar, one lookback interface to be advertised to the PE router and a small bit of BGP configuration. You could use different autonomous system (AS) numbers between customer routers, in this case I used the same AS for the same customer. In BGP this would introduce a loop since the router will see its own AS in the path. This design option adds the command of *allowas-in* to our BGP configuration. This feature allows a route to be received even if the routers own AS is in path.

## PE Routers Configuration Example

```text
interface Ethernet0/1
 ip address 10.0.15.1 255.255.255.0
 ip ospf network point-to-point
 mpls ip
!

router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.0.15.1 0.0.0.0 area 0
!
router bgp 12345
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 5.5.5.5 remote-as 12345
 neighbor 5.5.5.5 update-source Loopback0
 !
 address-family ipv4
 exit-address-family
 !
 address-family vpnv4
  neighbor 5.5.5.5 activate
  neighbor 5.5.5.5 send-community extended
 exit-address-family
 !
!
```

Internally we are running OSPF to give MPLS all the paths to work with. Every PE device has essentially the same iBGP configuration towards the P router. If you look closely we are not even bringing up the peering relationship with the IPv4 address family. Only the VPNv4 address family will be used. This is due to the BGP peering only being used to distribute the VPNv4 routes across the PE routers.

## P/RR Router Configuration Example

```text
interface Loopback0
 ip address 5.5.5.5 255.255.255.255
!
interface Ethernet0/0
 ip address 10.0.15.5 255.255.255.0
 ip ospf network point-to-point
 mpls ip
!
interface Ethernet0/1
 ip address 10.0.25.5 255.255.255.0
 ip ospf network point-to-point
 mpls ip
!
interface Ethernet0/2
 ip address 10.0.35.5 255.255.255.0
 ip ospf network point-to-point
 mpls ip
!
interface Ethernet0/3
 ip address 10.0.45.5 255.255.255.0
 ip ospf network point-to-point
 mpls ip
!
router ospf 1
 router-id 5.5.5.5
 network 0.0.0.0 255.255.255.255 area 0
!
router bgp 12345
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor iBGP peer-group
 neighbor iBGP remote-as 12345
 neighbor iBGP update-source Loopback0
 neighbor 1.1.1.1 peer-group iBGP
 neighbor 2.2.2.2 peer-group iBGP
 neighbor 3.3.3.3 peer-group iBGP
 neighbor 4.4.4.4 peer-group iBGP
 !
 address-family ipv4
 exit-address-family
 !
 address-family vpnv4
  neighbor iBGP send-community extended
  neighbor iBGP route-reflector-client
  neighbor 1.1.1.1 activate
  neighbor 2.2.2.2 activate
  neighbor 3.3.3.3 activate
  neighbor 4.4.4.4 activate
 exit-address-family
!
```

The P router configuration does look a bit more involved but in reality its not too bad. In the case of the P router, I just advertised all the things in OSPF for simplicity. As I mentioned earlier with the PE routers, only the VPNv4 peering will be used. I created a peer group to bring down the number of commands required to make this all work.

## Introduction to Nornir

I will break down the structure of the repository I made but I wont go too deep into details. Mainly because there’s a lot of great resources out there to check out (will link at the bottom of the post), and I want to try and keep this post some what short. Nornir is a network automation framework that is written in Python. This allows developers and engineers to build automation and tools with a very powerful programming language. This also allows individuals to easily extend the functionality of the framework to meet their needs.

`config.yaml`

```yaml
---
inventory:
  plugin: SimpleInventory
  options:
    host_file: "hosts.yaml"
    group_file: "groups.yaml"
runner:
  plugin: threaded
  options:
    num_workers: 10
```

The `config.yaml` file is pretty important in Nornir. In the sample above we are using the `SimpleInventory` plugin and specifying where the host and group files are located. I am using 10 workers but my lab is fairly small so I could have used less here.

`hosts.yaml`

```yaml
---
PE1:
  hostname: "192.168.10.176"
  groups:
    - pe
  data:
    interfaces:
      - name: Ethernet0/0
        vrf: CUSTOMER_789
        ip: 10.0.11.1/24
    bgp:
      neighbors:
        - vrf_name: CUSTOMER_789
          remote_ip: 10.0.11.2
          remote_as: 789
```

I included the setup for our PE1 router, the rest follow a similar structure. Anything host specific I set at the `host.yaml` file and anything group specific is set at the `groups.yaml` file you will see shortly. I assign all PE routers to the `pe` group and set certain interface parameters as well as BGP information that will be used in the future.

`groups.yaml`

```yaml
---
pe:
  platform: ios
  username: cisco
  password: cisco
  data:
    vrfs:
      - name: CUSTOMER_789
        rd: "789:1"
        rt_import: "789:1"
        rt_export: "789:1"

      - name: CUSTOMER_777
        rd: "777:1"
        rt_import: "777:1"
        rt_export: "777:1"
```

The groups file is pretty neat, since all of my routers are part of the pe group and have a similar platform as well as authentication parameters, I decided to include all of that information under the group. In this automation example I wanted all VRFs to be included on all PE routers. Therefore I also added the VRF information under the group file vs under the host data.

## Breaking Down Script

`l3vpn_deploy.py snippet`

```python
from nornir import InitNornir
from nornir_scrapli.tasks import send_configs
from nornir_jinja2.plugins.tasks import template_file
from nornir_napalm.plugins.tasks import napalm_get
from nornir_utils.plugins.tasks.files import write_file
from nornir_utils.plugins.functions import print_result
from net_utils import address, mask
```

The portion above is essentially importing any tasks or functions we will need to run Nornir. Whether its importing *InitNornir* or plugins used to connect to devices and send commands. To learn more about plugins that can be used with Nornir please check out the [nornir.tech](https://nornir.tech/nornir/plugins/) site.

`l3vpn_deploy.py snippet`

```python
def main():

    """
    Main that calls l3vpn function
    """

    nornir = InitNornir(config_file="config.yaml")
    print("Nornir initialized with the following hosts:\n")
    for host in nornir.inventory.hosts.keys():
        print(f"{host}\n")

    result = nornir.run(task=l3vpn)

    print_result(result)
```

This portion of the script is our `main` function and essentially kicks off all the other portions of our script to execute. Nornir is initialized with our `config.yaml` file that includes information about our hosts, groups, and connection parameters. There is a simple print statement at the bottom to let the operator know what inventory was just initialized for this job run. We will call on the `l3vpn` function that is defined towards the top of the script that I will walk through in a bit.

## Creating the L3VPNs

We will need the following items on our PE routers to complete our L3VPN build:

- Customer VRFs created with RD and RTs
- Assign VRF to interface facing customer router
- Configure BGP relationship between PE and CE device under new VRF

Now that we have the main portions required for our build, it was just a matter of converting the manual configurations into simple Jinja templates. I will spare you from reading about every task since they all follow the same pattern; create commands from Jinja template, split commands, send commands to PE routers. Here is a snippet below of what I mean.

`l3vpn_deploy.py snippet`

```python
def l3vpn(task):

    """
    Main function built for L3VPN deployment tasks
    """
    task1_result = task.run(
        name=f"{task.host.name}: Creating VRFs Configuration",
        task=template_file,
        template="vrf.j2",
        path="templates/",
        data=task.host["vrfs"],
    )
    vrf_config = task1_result[0].result

    task2_result = task.run(
        name=f"{task.host.name}: Configuring VRFs on PE Nodes",
        task=send_configs,
        configs=vrf_config.split("\n"),
    )
```

I name the tasks for clarity when seeing the job output. Initially we run a *template_file* task that will create commands from a template. Then its a matter of pointing it to our specific template and feeding it data. Once that is done I set the output of that task to a variable called *vrf_config*, this can then be fed into the next task to push the commands to our PE routers.

```jinja2
{% for vrf in data %}
vrf definition {{ vrf.name }}
 rd {{ vrf.rd }}
 route-target export {{ vrf.rt_export }}
 route-target import {{ vrf.rt_import }}
 !
 address-family ipv4
 exit-address-family
{% endfor %}
```

Example of the VRF Jinja template above. The rest of the items for L3VPNs use the same structure. Feel free to check out the git repository linked below to see the rest. Ill show the output from a few PE and CE routers before and after the changes.

```text
PE1#show vrf brief
  Name                             Default RD            Protocols   Interfaces
  MGMT                             <not set>             ipv4        Et0/3
PE1#
PE2#show vrf brief
  Name                             Default RD            Protocols   Interfaces
  MGMT                             <not set>             ipv4
PE2#
PE3#show vrf brief
  Name                             Default RD            Protocols   Interfaces
  MGMT                             <not set>             ipv4
PE3#
PE4#show vrf brief
  Name                             Default RD            Protocols   Interfaces
  MGMT                             <not set>             ipv4        Et0/3
PE4#
##### CE Devices #####
CE1#show ip bgp  summary | b Neigh
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.11.1       4        12345       0       0        1    0    0 00:10:51 Active
CE1#show ip bgp | b Network
     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.1.0/24    0.0.0.0                  0         32768 i
CE1#
CE2#show ip bgp summary | b Neigh
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.22.1       4        12345       0       0        1    0    0 00:13:22 Idle
CE2#show ip bgp | b Network
     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.2.0/24    0.0.0.0                  0         32768 i
CE2#
CE3#show ip bgp summary | b Neigh
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.33.1       4        12345       0       0        1    0    0 never    Active
CE3#show ip bgp | b Network
     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.1.0/24    0.0.0.0                  0         32768 i
CE3#
CE4#show ip bgp summary | b Neigh
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.44.1       4        12345       0       0        1    0    0 never    Active
CE4#show ip bgp | b Network
     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.2.0/24    0.0.0.0                  0         32768 i
CE4#
```

## Script Output

```python
(venv) [juliopdx@pbpro auto_mpls_l3vpn]$ time python l3vpn_deploy.py
Nornir initialized with the following hosts:

PE1

PE2

PE3

PE4

l3vpn***************************************************************************
* PE1 ** changed : True ********************************************************
vvvv l3vpn ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
---- PE1: Creating VRFs Configuration ** changed : False ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family

---- PE1: Configuring VRFs on PE Nodes ** changed : True ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family


---- PE1: Create VRF to Interfaces Configuration ** changed : False ------------ INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_789
 ip address 10.0.11.1 255.255.255.0
!

---- PE1: Configuring VRFs on Interfaces ** changed : True --------------------- INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_789
 ip address 10.0.11.1 255.255.255.0
!


---- PE1: Create BGP Neighbor Configuration ** changed : False ----------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_789
  neighbor 10.0.11.2 remote-as 789
  neighbor 10.0.11.2 activate
 exit-address-family

---- PE1: Configuring BGP Neighbors under VRFs ** changed : True --------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_789
  neighbor 10.0.11.2 remote-as 789
  neighbor 10.0.11.2 activate
 exit-address-family


^^^^ END l3vpn ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* PE2 ** changed : True ********************************************************
vvvv l3vpn ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
---- PE2: Creating VRFs Configuration ** changed : False ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family

---- PE2: Configuring VRFs on PE Nodes ** changed : True ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family


---- PE2: Create VRF to Interfaces Configuration ** changed : False ------------ INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_777
 ip address 10.0.22.1 255.255.255.0
!

---- PE2: Configuring VRFs on Interfaces ** changed : True --------------------- INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_777
 ip address 10.0.22.1 255.255.255.0
!


---- PE2: Create BGP Neighbor Configuration ** changed : False ----------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_777
  neighbor 10.0.22.2 remote-as 777
  neighbor 10.0.22.2 activate
 exit-address-family

---- PE2: Configuring BGP Neighbors under VRFs ** changed : True --------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_777
  neighbor 10.0.22.2 remote-as 777
  neighbor 10.0.22.2 activate
 exit-address-family


^^^^ END l3vpn ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* PE3 ** changed : True ********************************************************
vvvv l3vpn ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
---- PE3: Creating VRFs Configuration ** changed : False ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family

---- PE3: Configuring VRFs on PE Nodes ** changed : True ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family


---- PE3: Create VRF to Interfaces Configuration ** changed : False ------------ INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_777
 ip address 10.0.33.1 255.255.255.0
!

---- PE3: Configuring VRFs on Interfaces ** changed : True --------------------- INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_777
 ip address 10.0.33.1 255.255.255.0
!


---- PE3: Create BGP Neighbor Configuration ** changed : False ----------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_777
  neighbor 10.0.33.2 remote-as 777
  neighbor 10.0.33.2 activate
 exit-address-family

---- PE3: Configuring BGP Neighbors under VRFs ** changed : True --------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_777
  neighbor 10.0.33.2 remote-as 777
  neighbor 10.0.33.2 activate
 exit-address-family


^^^^ END l3vpn ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* PE4 ** changed : True ********************************************************
vvvv l3vpn ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
---- PE4: Creating VRFs Configuration ** changed : False ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family

---- PE4: Configuring VRFs on PE Nodes ** changed : True ----------------------- INFO
vrf definition CUSTOMER_789
 rd 789:1
 route-target export 789:1
 route-target import 789:1
 !
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER_777
 rd 777:1
 route-target export 777:1
 route-target import 777:1
 !
 address-family ipv4
 exit-address-family


---- PE4: Create VRF to Interfaces Configuration ** changed : False ------------ INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_789
 ip address 10.0.44.1 255.255.255.0
!

---- PE4: Configuring VRFs on Interfaces ** changed : True --------------------- INFO
interface Ethernet0/0
 vrf forwarding CUSTOMER_789
 ip address 10.0.44.1 255.255.255.0
!


---- PE4: Create BGP Neighbor Configuration ** changed : False ----------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_789
  neighbor 10.0.44.2 remote-as 789
  neighbor 10.0.44.2 activate
 exit-address-family

---- PE4: Configuring BGP Neighbors under VRFs ** changed : True --------------- INFO
router bgp 12345
  address-family ipv4 vrf CUSTOMER_789
  neighbor 10.0.44.2 remote-as 789
  neighbor 10.0.44.2 activate
 exit-address-family


^^^^ END l3vpn ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

real    0m9.216s
user    0m6.683s
sys     0m1.079s
(venv) [juliopdx@pbpro auto_mpls_l3vpn]$
```

## VRF Creation Validation

```text
PE1#show ip vrf
  Name                             Default RD            Interfaces
  CUSTOMER_777                     777:1
  CUSTOMER_789                     789:1                 Et0/0
  MGMT                             <not set>             Et0/3
PE1#
PE2#show ip vrf
  Name                             Default RD            Interfaces
  CUSTOMER_777                     777:1                 Et0/0
  CUSTOMER_789                     789:1
  MGMT                             <not set>
PE2#
PE3#show ip vrf
  Name                             Default RD            Interfaces
  CUSTOMER_777                     777:1                 Et0/0
  CUSTOMER_789                     789:1
  MGMT                             <not set>
PE3#
PE4#show ip vrf
  Name                             Default RD            Interfaces
  CUSTOMER_777                     777:1
  CUSTOMER_789                     789:1                 Et0/0
  MGMT                             <not set>             Et0/3
PE4#
```

## BGP Peers and Routes on CE Routers

```text
CE1#show ip bgp  summary | b Neigh
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.11.1       4        12345      19      20        3    0    0 00:13:17        1
CE1#show ip bgp | b Network
     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.1.0/24    0.0.0.0                  0         32768 i
 *>  172.16.2.0/24    10.0.11.1                              0 12345 789 i
CE1#traceroute 172.16.2.1 source 172.16.1.1 probe 1 numeric
Type escape sequence to abort.
Tracing the route to 172.16.2.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.11.1 0 msec
  2 10.0.15.5 [MPLS: Labels 300/24 Exp 0] 2 msec
  3 10.0.44.1 [MPLS: Label 24 Exp 0] 2 msec
  4 10.0.44.2 1 msec
CE1#
CE3#show ip bgp summary | b Neigh
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.33.1       4        12345      25      23        3    0    0 00:17:34        1
CE3#show ip bgp | b Network
     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.1.0/24    0.0.0.0                  0         32768 i
 *>  172.16.2.0/24    10.0.33.1                              0 12345 777 i
CE3#traceroute 172.16.2.1 source 172.16.1.1 probe 1 numeric
Type escape sequence to abort.
Tracing the route to 172.16.2.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.33.1 1 msec
  2 10.0.35.5 [MPLS: Labels 302/407 Exp 0] 2 msec
  3 10.0.22.1 [MPLS: Label 407 Exp 0] 2 msec
  4 10.0.22.2 2 msec
CE3#
```

## Outro and Links

I think there’s always room for improvement. Even off the top of my head a few additions could be the following:

- Automated testing
- Validation with something like pyATS to validate VRF creation on PE routers

I hope you found this a bit useful and maybe inspires you to build something you can use in everyday workload. Feel free to check out the links below to a few resources I found very helpful. Thank you again for reading this far, really means a lot.

- [GitHub Repository](https://github.com/JulioPDX/auto_mpls_l3vpn)
- [Nornir Documentation](https://nornir.readthedocs.io/en/latest/)
- [Nornir Plugins](https://nornir.tech/nornir/plugins/)
- [Automating Networks with Python by Nick Russo](https://www.pluralsight.com/courses/automating-networks-python)
- [Nornir Blog Post by Said van de Klundert](https://saidvandeklundert.net/2020-12-06-nornir/)
