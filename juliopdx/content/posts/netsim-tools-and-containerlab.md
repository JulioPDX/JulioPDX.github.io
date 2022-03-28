---
title: "Network Simulation Tools and Containerlab"
date: 2022-02-13T21:07:43-08:00
draft: false
toc: true
mermaid: true
images:
  - /blog/images/philip-swinburn.jpg
tags:
  - Containerlab
  - OSPF
  - BGP
  - net-sim
  - Docker
  - Ansible
---

## Introduction

I've been progressing through a series of technical books, some of which I've shared on other blogs. A few of them focus on BGP. BGP being so broad, I decided to create a [challenge lab](https://github.com/JulioPDX/learning_labs/tree/main/labs/bgp/arista). Creating the challenge/troubleshooting labs has really made more concepts stick. I'm trying to use the principle of teaching someone to make the learning last. These have been incredibly fun to create and the community interaction has been amazing. One brave soul(Jeroen van Bemmel) shared his [solution](https://github.com/jbemmel/netsim-examples/tree/evpn-vxlan-to-hosts/BGP/Julio-Challenge). I was fascinated on his solution and how he created his topology with Containerlab and net-sim tools.

## net-sim tools

Network Simulation Tools was created by Ivan Pepelnjak of [ipspace.net](https://www.ipspace.net/Main_Page). I think Ivan was tired of the point and click required when making a lab topology with other tools. The aim of the tool is to take infrastructure as code principles and leverage them for the creation of network topologies. Topologies can be defined in a simple YAML file and net-sim will take care of link connections, addressing(IPv4/6), routing protocols, etc...

Usually when creating a lab, there is a slow process of starting up nodes, connecting links, selecting address space, configuring hostname Rx, configuring ip add 10.x.x.x 255.255.255.0, no shut, and so on and so forth. At this point, I don't think I need to configure an IP address on an interface to know I've mastered that skill :).

### Getting Started

In this post we will leverage Containerlab with net-sim tools and below are the requirements to get started.

- [Install Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)
- [Install Containerlab](https://containerlab.srlinux.dev/install/)
- [Import Docker Container Images](https://juliopdx.com/2021/12/10/my-journey-and-experience-with-containerlab/)

There are a few options to get started with net-sim tools, but I think using a python virtualenv was fairly simple.

```shell
python3 -m venv venv
source venv/bin/activate
pip install netsim-tools
pip install ansible #decided to use whatever was latest and it works just fine for me
```

I know this seems like a lot but the benefits are definitely worth it.

### Defining a Topology

Let me share a basic topology and walk you through what will be created. Please note, some of the options I have defined are already defaults, I just decided to add them to the topology file for reader clarity.

```yaml
name: basic
provider: clab
defaults:
  device: eos
  devices:
    eos.clab.image: ceos:4.27.2F


addressing:
  loopback:
    ipv4: 10.0.0.0/24
    ipv6: 200::/7
  router_id:
    ipv4: 10.0.0.0/24
    prefix: 32
  p2p:
    ipv4: 10.1.0.0/16
    prefix: 30
    ipv6: true

nodes: [R1, R2, R3]

links:
  - R1-R2
  - R2-R3

```

This Topology will perform the following:

- Use Containerlab to deploy our nodes
- Sets a default image version for EOS devices(saves a lot of typing)
- Set prefix information for different connections, not required since net-sim will do this for you.
- Define how many nodes we want
- Define how we want the nodes connected.

A few things to note. I defined the IP space but that is not required when creating lab topologies. No need to worry about interface names, the logic within net-sim tools will just increment over the next available interface.

#### Providers

At the time of this writing, net-sim tools supports three popular providers; vagrant-libvirt, Vagrant VirtualBox, and Containerlab. As long as the user is familiar with their provider of choice, the interaction with net-sim tools is similar across the board. In the example topology file above, we defined Containerlab as our provider. This will cause net-sim tools to create a `clab.yml` file in the structure Containerlab is used to. This `clab.yml` file will then be used to deploy our network devices.

#### Images

Net-sim does not come pre packaged with any network operating systems(NOS) images. A few vendors have made their images publicly available(see Nokia/FRR) as a simple docker pull. For any other images, please see their documentation to obtain an image. For this example, I will be using some Arista cEOS nodes. They can be obtained by signing up to their support site(free).

#### Networks

Net-sim gives you pretty much any option you could want for network addressing. You can either use built-in address pools, assign statics, or define your own pools. Net-sim will then handle the IP addressing for each network device and interface. Very simple example below, please check out their in-depth [addressing tutorial](https://netsim-tools.readthedocs.io/en/latest/example/addressing-tutorial.html)!

```yml
  p2p:
    ipv4: 10.1.0.0/16
    prefix: 30
    ipv6: true

nodes: [R1, R2, R3]

links:
  - R1-R2
  - R2-R3
```

{{< mermaid >}}
graph LR
  R1((R1)) --- R2((R2))
  R2 --- R3((R3))
{{< /mermaid >}}

Here we have three nodes with two point to point links. R1 will have node ID 1, R2 will be node ID 2, etc... The first `/30` prefix in the `10.1.0.0/16` network will be `10.1.0.0-3`. This will be assigned to the link between R1-R2. The next available `/30` will be `10.1.0.4-7`. Same as before, this network will be assigned to the R2-R3 link.

### Deployments and How It All Works

The CLI has a few neat options, let me show you some I think will be used the most.

`netlab create`

This command will create any files necessary to deploy your containers and configure devices with Ansible. Great if you would just like to view the data without actually performing any deployments.

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ netlab create basic.yml
Created provider configuration file: clab.yml
Created group_vars for all
Created group_vars for eos
Created host_vars for R1
Created host_vars for R2
Created host_vars for R3
Created minimized Ansible inventory hosts.yml
Created Ansible configuration file: ansible.cfg
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$
```

Lets take a look at the `clab.yml` file.

```yml
name: basic

topology:
  nodes:
    R1:
      kind: ceos
      image: ceos:4.27.2F

    R2:
      kind: ceos
      image: ceos:4.27.2F

    R3:
      kind: ceos
      image: ceos:4.27.2F



  links:
  - endpoints:
    - "R1:eth1"
    - "R2:eth1"
  - endpoints:
    - "R2:eth2"
    - "R3:eth1"

```

This can then be used to run either `netlab up basic.yml` or `sudo containerlab deploy -t clab.yml`. One of the more interesting files created is the `host_vars/<device name>/topology.yml`. This gives you a great look at everything that will be deployed on the device with Ansible

```yml
# Ansible inventory created from ['basic.yml', 'package:topology-defaults.yml']
#
---
af:
  ipv4: true
  ipv6: true
box: ceos:4.27.2F
clab:
  kind: ceos
hostname: clab-basic-R1
interfaces:
- clab:
    name: eth1
  ifindex: 1
  ifname: Ethernet1
  ipv4: 10.1.0.1/30
  ipv6: true
  linkindex: 1
  name: R1 -> R2
  neighbors:
  - ifname: Ethernet1
    ipv4: 10.1.0.2/30
    ipv6: true
    node: R2
  remote_id: 2
  remote_ifindex: 1
  type: p2p
loopback:
  ipv4: 10.0.0.1/32
  ipv6: 200:0:0:1::1/64
mgmt:
  ifname: Management0
  ipv4: 192.168.121.101
  mac: 08-4F-A9-00-00-01

```

`netlab up`

This command will actually deploy our nodes with Containerlab and configure the initial configuration with Ansible. You can use this command instead of using individual commands to deploy the topology.

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ netlab up basic.yml
Created provider configuration file: clab.yml
Created group_vars for all
Created group_vars for eos
Created host_vars for R1
Created host_vars for R2
Created host_vars for R3
Created minimized Ansible inventory hosts.yml
Created Ansible configuration file: ansible.cfg

Step 2: Checking virtualization provider installation
============================================================
.. all tests succeeded, moving on

Step 3: starting the lab
============================================================
INFO[0000] Containerlab v0.23.0 started
INFO[0000] Parsing & checking topology file: clab.yml
INFO[0000] Creating lab directory: /home/juliopdx/repos/cl-nst-demo/clab-basic
INFO[0000] Creating docker network: Name='clab', IPv4Subnet='172.20.20.0/24', IPv6Subnet='2001:172:20:20::/64', MTU='1500'
INFO[0000] Creating container: R2
INFO[0000] Creating container: R1
INFO[0000] Creating container: R3
INFO[0001] Creating virtual wire: R1:eth1 <--> R2:eth1
INFO[0001] Creating virtual wire: R2:eth2 <--> R3:eth1
INFO[0001] Running postdeploy actions for Arista cEOS 'R1' node
INFO[0001] Running postdeploy actions for Arista cEOS 'R2' node
INFO[0001] Running postdeploy actions for Arista cEOS 'R3' node
INFO[0053] Adding containerlab host entries to /etc/hosts file
+---+---------------+--------------+--------------+------+---------+----------------+----------------------+
| # |     Name      | Container ID |    Image     | Kind |  State  |  IPv4 Address  |     IPv6 Address     |
+---+---------------+--------------+--------------+------+---------+----------------+----------------------+
| 1 | clab-basic-R1 | ffb8857ca087 | ceos:4.27.2F | ceos | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 2 | clab-basic-R2 | c533fc2411e0 | ceos:4.27.2F | ceos | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 3 | clab-basic-R3 | da9fedb7ce7f | ceos:4.27.2F | ceos | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
+---+---------------+--------------+--------------+------+---------+----------------+----------------------+

Step 4: deploying initial device configurations
============================================================

PLAY [Deploy device configuration] *************************************************************

TASK [Find initial configuration template] *****************************************************
skipping: [R1] => (item=/home/juliopdx/repos/cl-nst-demo/venv/lib/python3.8/site-packages/netsim/ansible/templates/initial/eos.j2)
skipping: [R1]
skipping: [R3] => (item=/home/juliopdx/repos/cl-nst-demo/venv/lib/python3.8/site-packages/netsim/ansible/templates/initial/eos.j2)
skipping: [R3]
skipping: [R2] => (item=/home/juliopdx/repos/cl-nst-demo/venv/lib/python3.8/site-packages/netsim/ansible/templates/initial/eos.j2)
skipping: [R2]

TASK [Deploy initial device configuration] *****************************************************
included: /home/juliopdx/repos/cl-nst-demo/venv/lib/python3.8/site-packages/netsim/ansible/tasks/deploy-config/eos.yml for R2, R1, R3 => (item=/home/juliopdx/repos/cl-nst-demo/venv/lib/python3.8/site-packages/netsim/ansible/tasks/deploy-config/eos.yml)

TASK [wait_for_connection] *********************************************************************
ok: [R2]
ok: [R1]
ok: [R3]

TASK [arista.eos.eos_config] *******************************************************************
[WARNING]: To ensure idempotency and correct diff the input configuration lines should be
similar to how they appear if present in the running configuration on device including the
indentation
changed: [R3]
changed: [R1]
changed: [R2]

TASK [Deploy module-specific configurations] ***************************************************

TASK [Deploy custom deployment templates] ******************************************************

PLAY RECAP *************************************************************************************
R1                         : ok=3    changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
R2                         : ok=3    changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
R3                         : ok=3    changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0

(venv) juliopdx@containerlab:~/repos/cl-nst-demo$
```

Checking the configuration on R1.

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ docker exec -it clab-basic-R1 Cli
R1>en
R1#show run
! Command: show running-config
! device: R1 (cEOSLab, EOS-4.27.2F-26069621.4272F (engineering build))
!
no aaa root
!
username admin privilege 15 role network-admin secret sha512 $6$KKU2DJRmdlpXm3wG$dZKDhT4eC5ZcNcwyPtklQfNrOnUw28P4dNYTSDB5LiBp/gV0r24ET/D.4L9AeBLmCwcwyMsgtopRkaHigNkHC.
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname R1
ip host R2 10.0.0.2 10.1.0.2 10.1.0.5
ip host R3 10.0.0.3 10.1.0.6
!
spanning-tree mode mstp
!
management api http-commands
   no shutdown
!
management api gnmi
   transport grpc default
!
management api netconf
   transport ssh default
!
interface Ethernet1
   description R1 -> R2
   mac-address 52:dc:ca:fe:01:01
   no switchport
   ip address 10.1.0.1/30
   ipv6 enable
!
interface Loopback0
   ip address 10.0.0.1/32
   ipv6 address 200:0:0:1::1/64
!
interface Management0
   ip address 172.20.20.2/24
   ipv6 address 2001:172:20:20::2/64
   no lldp transmit
   no lldp receive
!
ip routing
!
ipv6 unicast-routing
!
end
R1#
```

I think that is pretty awesome. With one command we have deployed nodes, set base parameters, and interface configurations. At this point someone could proceed to learn whatever the objective is for the day or test some design.

`netlab down`

This command will remove all containers and remove any directory created by Containerlab(startup configs). If you would like to keep startup configurations, I would recommend you shutdown the lab with Containerlab commands.

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ netlab down basic.yml

Step 1: Checking virtualization provider installation
============================================================
.. all tests succeeded, moving on


Step 2: stopping the lab
============================================================
INFO[0000] Parsing & checking topology file: clab.yml
INFO[0000] Destroying lab: basic
INFO[0001] Removed container: clab-basic-R1
INFO[0001] Removed container: clab-basic-R3
INFO[0001] Removed container: clab-basic-R2
INFO[0001] Removing containerlab host entries from /etc/hosts file
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$
```

#### Python, Ansible, and Jinja

Behind the scenes Python is wrapping everything that has been defined into usable data. This can then be used with Ansible and Jinja to create vendor specific configurations. This could be interface settings, BGP, OSPF, and others. For a list of supported configurations by NOS, check it out [here](https://netsim-tools.readthedocs.io/en/latest/platforms.html#supported-configuration-modules). I have probably atrociously oversimplified this process, thankfully the creators made this [in-depth guide](https://netsim-tools.readthedocs.io/en/latest/dev/transform.html) on the process.

### Sample Use Cases

So far I think this is pretty neat. Similar to Containerlab, I think net-sim lends itself incredibly well in learning environments or testing solutions/designs. Let me walk you through some simple workflows someone might go through when learning a technology.

#### Basic Lab for OSPF Learning

We have shown this topology already but bear with me. In this example we have three nodes that we would like to deploy to start learning some OSPF. The topology file can deploy nodes and configure all interface settings.

```yml
name: basic
provider: clab
defaults:
  device: eos
  devices:
    eos.clab.image: ceos:4.27.2F


addressing:
  loopback:
    ipv4: 10.0.0.0/24
    ipv6: 200::/7
  router_id:
    ipv4: 10.0.0.0/24
    prefix: 32
  p2p:
    ipv4: 10.1.0.0/16
    prefix: 30
    ipv6: true

nodes: [R1, R2, R3]

links:
  - R1-R2
  - R2-R3

```

#### Multi Area OSPF

Maybe we want to deploy an OSPF topology with multiple areas. This could be used to get the concepts of IA vs O vs E vs N routes down. This file will deploy R2 in area 0. R1 will be in area 1, but the link between R1 and R2 will be in area 0. R3 will be in area 3, but the link between R2 and R3 will be in area 0.

```yml
name: ospf
provider: clab
defaults:
  device: eos
  devices:
    eos.clab.image: ceos:4.27.2F


addressing:
  loopback:
    ipv4: 10.0.0.0/24
    ipv6: 200::/7
  router_id:
    ipv4: 10.0.0.0/24
    prefix: 32
  p2p:
    ipv4: 10.1.0.0/16
    prefix: 30
    ipv6: true

module: [ospf]
ospf:
  area: 0

nodes:
  - name: R1
    ospf:
      area: 1
  - name: R2
  - name: R3
    ospf:
      area: 3

links:
  - R1:
      ospf.area: 0
    R2:
  - R2:
    R3:
      ospf.area: 0

```

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ netlab up ospf.yml
Created provider configuration file: clab.yml
Created group_vars for all
Created group_vars for eos
Created host_vars for R1
Created host_vars for R2
Created host_vars for R3
Created minimized Ansible inventory hosts.yml
Created Ansible configuration file: ansible.cfg

Step 2: Checking virtualization provider installation
============================================================
.. all tests succeeded, moving on

Step 3: starting the lab
============================================================
INFO[0000] Containerlab v0.23.0 started
INFO[0000] Parsing & checking topology file: clab.yml
INFO[0000] Creating lab directory: /home/juliopdx/repos/cl-nst-demo/clab-ospf
INFO[0000] Creating docker network: Name='clab', IPv4Subnet='172.20.20.0/24', IPv6Subnet='2001:172:20:20::/64', MTU='1500'
INFO[0000] Creating container: R1
INFO[0000] Creating container: R2
INFO[0000] Creating container: R3
INFO[0001] Creating virtual wire: R2:eth2 <--> R3:eth1
INFO[0001] Creating virtual wire: R1:eth1 <--> R2:eth1
INFO[0001] Running postdeploy actions for Arista cEOS 'R3' node
INFO[0001] Running postdeploy actions for Arista cEOS 'R2' node
INFO[0001] Running postdeploy actions for Arista cEOS 'R1' node
INFO[0054] Adding containerlab host entries to /etc/hosts file
+---+--------------+--------------+--------------+------+---------+----------------+----------------------+
| # |     Name     | Container ID |    Image     | Kind |  State  |  IPv4 Address  |     IPv6 Address     |
+---+--------------+--------------+--------------+------+---------+----------------+----------------------+
| 1 | clab-ospf-R1 | ba1c4a47b6cb | ceos:4.27.2F | ceos | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 2 | clab-ospf-R2 | dbda30424bc2 | ceos:4.27.2F | ceos | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 3 | clab-ospf-R3 | 3234e3265e9c | ceos:4.27.2F | ceos | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
+---+--------------+--------------+--------------+------+---------+----------------+----------------------+

etc...
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$
```

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ docker exec -it clab-ospf-R1 Cli
R1>en
R1#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.2        1        default  0   FULL                   00:00:38    10.1.0.2        Ethernet1
R1#show ip route ospf

VRF: default

 O        10.0.0.2/32 [110/20] via 10.1.0.2, Ethernet1
 O IA     10.0.0.3/32 [110/30] via 10.1.0.2, Ethernet1
 O        10.1.0.4/30 [110/20] via 10.1.0.2, Ethernet1

R1#
```

#### OSPF Lab for BGP Learning

In this topology, the lab is predeployed with OSPF(area 0) and the students could then focus on the BGP learning.

```yml
name: basico
provider: clab
defaults:
  device: eos
  devices:
    eos.clab.image: ceos:4.27.2F


addressing:
  loopback:
    ipv4: 10.0.0.0/24
    ipv6: 200::/7
  router_id:
    ipv4: 10.0.0.0/24
    prefix: 32
  p2p:
    ipv4: 10.1.0.0/16
    prefix: 30
    ipv6: true

module: [ospf]

nodes: [R1, R2, R3]

links:
  - R1-R2
  - R2-R3

```

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ netlab up basic_ospf.yml
Created provider configuration file: clab.yml
Created group_vars for all
Created group_vars for eos
Created host_vars for R1
Created host_vars for R2
Created host_vars for R3
Created minimized Ansible inventory hosts.yml
Created Ansible configuration file: ansible.cfg

Step 2: Checking virtualization provider installation
============================================================
.. all tests succeeded, moving on

Step 3: starting the lab
============================================================
INFO[0000] Containerlab v0.23.0 started
INFO[0000] Parsing & checking topology file: clab.yml
INFO[0000] Creating lab directory: /home/juliopdx/repos/cl-nst-demo/clab-basico
INFO[0000] Creating docker network: Name='clab', IPv4Subnet='172.20.20.0/24', IPv6Subnet='2001:172:20:20::/64', MTU='1500'
INFO[0000] Creating container: R2
INFO[0000] Creating container: R1
INFO[0000] Creating container: R3
INFO[0001] Creating virtual wire: R1:eth1 <--> R2:eth1
INFO[0001] Creating virtual wire: R2:eth2 <--> R3:eth1
INFO[0001] Running postdeploy actions for Arista cEOS 'R3' node
INFO[0001] Running postdeploy actions for Arista cEOS 'R2' node
INFO[0001] Running postdeploy actions for Arista cEOS 'R1' node
INFO[0055] Adding containerlab host entries to /etc/hosts file
+---+----------------+--------------+--------------+------+---------+----------------+----------------------+
| # |      Name      | Container ID |    Image     | Kind |  State  |  IPv4 Address  |     IPv6 Address     |
+---+----------------+--------------+--------------+------+---------+----------------+----------------------+
| 1 | clab-basico-R1 | f079a253b1d1 | ceos:4.27.2F | ceos | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 2 | clab-basico-R2 | ed28240d8b0f | ceos:4.27.2F | ceos | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 3 | clab-basico-R3 | 0f643cc95fc7 | ceos:4.27.2F | ceos | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
+---+----------------+--------------+--------------+------+---------+----------------+----------------------+
etc...
```

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ docker exec -it clab-basico-R1 Cli
R1>en
R1#show ip route ospf

VRF: default

 O        10.0.0.2/32 [110/20] via 10.1.0.2, Ethernet1
 O        10.0.0.3/32 [110/30] via 10.1.0.2, Ethernet1
 O        10.1.0.4/30 [110/20] via 10.1.0.2, Ethernet1

R1#
```

#### OSPF and BGP Lab Complete

In this topology, we have a complete OSPF deployment. R1 and R3 will be iBGP peers.

{{< mermaid >}}
%%{init: {'securityLevel': 'loose', 'theme':'dark'}}%%
flowchart TD
  subgraph OSPF
    subgraph AS 65000
        1((R1)) -.iBGP.- 3((R3))
    end
  1 --- 2((R2))
  3 --- 2((R2))
  end
{{</ mermaid >}}

```yml
name: bgp
provider: clab
defaults:
  device: eos
  devices:
    eos.clab.image: ceos:4.27.2F


addressing:
  loopback:
    ipv4: 10.0.0.0/24
    ipv6: 200::/7
  router_id:
    ipv4: 10.0.0.0/24
    prefix: 32
  p2p:
    ipv4: 10.1.0.0/16
    prefix: 30
    ipv6: true

module: [ospf]

nodes:
  - name: R1
    module: [ospf,bgp]
    bgp.as: 65000
  - R2
  - name: R3
    module: [ospf,bgp]
    bgp.as: 65000

links:
  - R1-R2
  - R2-R3

```

```shell
(venv) juliopdx@containerlab:~/repos/cl-nst-demo$ docker exec -it clab-bgp-R1 Cli
R1>en
R1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  R3                       10.0.0.3 4 65000             10        10    0    0 00:05:32 Estab   1      1
R1#show ip bgp | b Network
          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            -                     -       -          -       0       i
 * >      10.0.0.3/32            10.0.0.3              0       -          100     0       i
R1#
```

## Outro and Links

Thank you all for reading this far. I really appreciate it! I am really looking forward to using net-sim tools to create future labs. I can already see the speed to create troubleshooting labs will be increased tremendously. Thank you, Ivan and team for all the work on this tool!

- [Featured Image Philip Swinburn](https://unsplash.com/photos/vS7LVkPyXJU)
- [net-sim tools](https://netsim-tools.readthedocs.io/en/latest/index.html)
- [Containerlab](https://containerlab.srlinux.dev/)
- [GitHub Repository](https://github.com/JulioPDX/cl-nst-demo)
