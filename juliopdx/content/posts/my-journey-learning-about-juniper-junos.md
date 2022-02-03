---
title: "My Journey Learning About Juniper Junos"
date: 2021-12-06T13:57:09-08:00
draft: false
toc: true
images:
tags:
  - Junos
  - Network Automation
  - Python
---

## Introduction

Hello and thank you for checking out another blog post. I’ve been under the weather for a bit but I am back on the keys. This post will be all about some neat things I discovered while learning a bit about the Juniper Junos NOS (network operating system). Previously I had never used Juniper, it was a vendor that always eluded me. My previous employers did not utilize this NOS and I never got around to adding it in the lab. Sit back and enjoy, I’ll include a neat automation example at the end.

## What is Junos?

Junos is the main operating system used in Juniper network devices. This could be the MX (routing), QFX (switching), and SRX (firewall). Since the same NOS is used across the product line, moving between them all in a configuration feels like being in the same device. This has a lot of benefits like eliminating that mode in your head of “which CLI am I in or which version of OS am I running?”. For the most part, it doesn’t matter, one configuration setting can transfer across all devices. At this point I’m probably overselling it, I’m sure there are some differences but you get the point.

## Reading the Junos CLI

When I first saw Junos CLI output… I’ll be honest, the first thought in my head was “WTF is that?”. Is it JSON? Is it a weird version of JSON? Most of my past experience was on the Cisco CLI and going from Cisco to this was a bit of a surprise. Below is a snippet from a Junos configuration (modified for brevity).

`Junos Config Snippet`

```text
juliopdx@R1> show configuration
## Last commit: 2021-12-06 11:19:02 PST by juliopdx
version 18.2R1.9;
system {
    login {
        message "Welcome to the world of tomorrow!";
    }
    root-authentication {
        encrypted-password "$6$xUmeLj6B$NBFrTm."; ## SECRET-DATA
    }
    host-name R1;
    time-zone America/Los_Angeles;
    services {
        ssh;
    }
}
interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                address 10.0.12.1/24;
            }
        }
    }
}
routing-options {
    static {
        route 0.0.0.0/0 {
            next-hop 192.168.10.1;
            no-readvertise;
        }
    }
    router-id 1.1.1.1;
}
protocols {
    ospf {
        reference-bandwidth 1g;
        area 0.0.0.0 {
            interface ge-0/0/0.0;
        }
    }
}
```

Looks a bit odd right? In reality, everything is in a nice orderly place. For example, any **system** configurations are neatly nested under that block. Any interface based settings would be under the **interfaces** block. Anything for OSPF is nested under **protocols ospf**.

## CLI Modes

I think I’m getting a bit ahead of myself. Lets talk about the CLI modes. Junos mainly uses two CLI modes; operational and configuration. The mode you see displayed above is operational mode. Think of this as the mode you would use to view items or perform some… wait for it… operational tasks (monitor, troubleshoot, connectivity… ping). The second mode, configuration, is used to perform any and all configuration tasks on a Junos device.

`Configuration Mode`

```text
juliopdx@R1> configure
Entering configuration mode

[edit]
juliopdx@R1# "Hi this is configuration mode"
```

Earlier I ran **show configuration** to view the configuration in operational mode. These commands can be executed in configuration mode as well by prepending **run** to the command. Please note, **show** works a bit differently when in configuration mode. Typing **show** will simply display the current configuration tree we are in (hold that thought for just a moment).

`Using ping with run`

```text
juliopdx@R1# run ping 10.0.23.3
PING 10.0.23.3 (10.0.23.3): 56 data bytes
64 bytes from 10.0.23.3: icmp_seq=0 ttl=64 time=331.137 ms
64 bytes from 10.0.23.3: icmp_seq=1 ttl=64 time=28.864 ms
64 bytes from 10.0.23.3: icmp_seq=2 ttl=64 time=697.261 ms
64 bytes from 10.0.23.3: icmp_seq=3 ttl=64 time=1142.495 ms
64 bytes from 10.0.23.3: icmp_seq=4 ttl=64 time=148.471 ms
^C
--- 10.0.23.3 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 28.864/469.646/1142.495/405.256 ms

[edit]
juliopdx@R1#
```

## Configuring Junos

At some point now we need to actually configure something. Let me just show you a small snippet of an interface configuration and we can walk through how the configuration goes.

```text
interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                address 10.0.12.1/24;
            }
        }
    }
```

When you want to configure or remove something in Junos, the commands will start with either **set** or **delete**. If we wanted to configure interface ge-0/0/0 with IP address 10.0.12.1/24, the command below would do just that.

```text
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/24
```

If you compare the configuration output to the command, its so similar, its basically the command in a tree format. You probably saw this and thought “What the heck is unit 0 and inet?”. Well **unit 0** will be used to create a sub-interface on the interface. Think of this as the default sub-interface that is created or all interfaces. Kind of how there is a secret default VRF in Cisco land but its not really displayed. The **inet** is essentially the IPv4 address family. Can you guess what **inet6** is?

That command got a bit long and there is an option to shorten our commands. Lets say for example we were about to drop a long configuration stanza on interface ge-0/0/0 and we wanted to shorten the amount of typing. We can actually tell Junos what section of the configuration we would like to edit. Example below.

```text
juliopdx@R1# edit interfaces ge-0/0/0 unit 0 family inet

[edit interfaces ge-0/0/0 unit 0 family inet]
juliopdx@R1#
```

At this point we are in configuration mode and in the ge-0/0/0 interface hierarchy. Remember when I said hold that though on the **show** command? If we type **show** now, it will only display what section we are currently on. Another example below.

```text
juliopdx@R1# show
address 10.0.12.1/24;

[edit interfaces ge-0/0/0 unit 0 family inet]
juliopdx@R1
```

Now if we wanted to configure that address, the command would be much shorter.

```text
juliopdx@R1# set address 10.0.12.1/24

[edit interfaces ge-0/0/0 unit 0 family inet]
juliopdx@R1#
```

## Replace, Compare, and Commit

I really like how Junos does this. If we were to make a change in Junos, it doesn’t immediately activate the change. It stores this in something called a candidate configuration. Lets test this. Currently interface ge-0/0/0 has an IP of 10.0.12.1, the remote side has an address of 10.0.12.2. Lets change the config and see if we can ping the remote side.

```text
[edit interfaces ge-0/0/0 unit 0 family inet]
juliopdx@R1# top

[edit]
juliopdx@R1# replace pattern 10.0.12.1/24 with 10.0.50.1/24

[edit]
juliopdx@R1#
```

I’ll walk through what is going on here. We entered **top** to return us to the highest level of the config hierarchy. We then execute another way of implementing a change, by using **replace**! This allows us to change a text patter with another, without stepping through the entire configuration. Lets check out the current state of our changes.

```text
juliopdx@R1# show | compare
[edit interfaces ge-0/0/0 unit 0 family inet]
+       address 10.0.50.1/24;
-       address 10.0.12.1/24;

[edit]
juliopdx@R1# run ping 10.0.12.2
PING 10.0.12.2 (10.0.12.2): 56 data bytes
64 bytes from 10.0.12.2: icmp_seq=0 ttl=64 time=269.266 ms
64 bytes from 10.0.12.2: icmp_seq=1 ttl=64 time=640.029 ms
64 bytes from 10.0.12.2: icmp_seq=2 ttl=64 time=116.563 ms
64 bytes from 10.0.12.2: icmp_seq=3 ttl=64 time=3.603 ms
^C
--- 10.0.12.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.603/257.365/640.029/240.205 ms

[edit]
juliopdx@R1#
```

At this point we can see that 10.0.12.1 has been replaced with 10.0.50.1 under the ge-0/0/0 interface. But we can still ping the remote side? This is because the change is only on the candidate configuration and not the running configuration. I’ll commit these changes and we should lose access to the remote side.


```text
juliopdx@R1# commit comment "This is a bad config"
commit complete

[edit]
juliopdx@R1# run ping 10.0.12.2
PING 10.0.12.2 (10.0.12.2): 56 data bytes
36 bytes from 4.68.38.157: Destination Net Unreachable
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
 4  5  00 0054 45fd   0 0000  3c  01 5773 192.168.10.143  10.0.12.2

36 bytes from 4.68.38.157: Destination Net Unreachable
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
 4  5  00 0054 4637   0 0000  3c  01 5739 192.168.10.143  10.0.12.2

36 bytes from 4.68.38.157: Destination Net Unreachable
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
 4  5  00 0054 464e   0 0000  3c  01 5722 192.168.10.143  10.0.12.2

^C
--- 10.0.12.2 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss

[edit]
juliopdx@R1#
```

## Rollback and Commit Confirmed

Clearly our change was not successful. Imagine this but at a grander scale than just my simple address change. Junos has a very simple option to go back to a previously working configuration. You may have noticed that we added a comment to our bad configuration change. Lets check out our previous commits.

`Previous Commits`

```text
[edit]
juliopdx@R1# run show system commit
0   2021-12-06 14:32:57 PST by juliopdx via cli
    This is a bad config
1   2021-12-06 13:41:27 PST by juliopdx via netconf
```

I’m staying in configuration mode so some of these commands are prepended with **run**. It looks like commit 0, the latest commit is our bad commit. Commit 1 is our last known good state. Lets roll this back and see if that gets us operational.

```text
juliopdx@R1# rollback 1
load complete

[edit]
juliopdx@R1# show interfaces ge-0/0/0
unit 0 {
    family inet {
        address 10.0.12.1/24;
    }
}

[edit]
juliopdx@R1# commit
commit complete

[edit]
juliopdx@R1# run ping 10.0.12.2
PING 10.0.12.2 (10.0.12.2): 56 data bytes
64 bytes from 10.0.12.2: icmp_seq=0 ttl=63 time=338.004 ms
64 bytes from 10.0.12.2: icmp_seq=1 ttl=63 time=8.134 ms
64 bytes from 10.0.12.2: icmp_seq=2 ttl=63 time=6.880 ms
64 bytes from 10.0.12.2: icmp_seq=3 ttl=63 time=13.623 ms
^C
--- 10.0.12.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 6.880/91.660/338.004/142.249 ms

[edit]
juliopdx@R1#
```

Well that was easy! Another option for safety is commit confirmed. Basically, if you run commit confirmed, you will need to run commit again to make sure you want to implement the change. This is great in case you cut yourself off from the remote device. Example below.

```text
juliopdx@R1# replace pattern 10.0.12.1/24 with 10.0.50.1/24

[edit]
juliopdx@R1# commit confirmed 1
commit confirmed will be automatically rolled back in 1 minutes unless confirmed
commit complete

# commit confirmed will be rolled back in 1 minute
[edit]
juliopdx@R1#
Broadcast Message from root@R1
        (no tty) at 14:44 PST...

Commit was not confirmed; automatic rollback complete.


[edit]
juliopdx@R1#
```

Pretty neat right? Since we did not type **commit** again, the change was automatically discarded.

## Viewing Configuration as Set Commands

There is another really neat way of viewing what has been configured on the device, **display set**. Imagine every **set** command that has been executed on the device being stored in a nice format that can be reused again. Please see snippet below.

```text
[edit]
juliopdx@R1# run show configuration | display set
set interfaces ge-0/0/0 unit 0 family inet address 10.0.12.1/24
set interfaces ge-0/0/1 unit 0 family inet address 10.0.13.1/24
set interfaces ge-0/0/2 unit 0 family inet address 10.0.0.1/24
set interfaces fxp0 unit 0 family inet address 192.168.10.143/24
set interfaces lo0 unit 0 family inet address 1.1.1.1/32
set snmp description somedevice
set snmp location "123 fake street"
set snmp contact "juliopdx@example.com"
set snmp community juliopdx authorization read-only
set snmp trap-group test categories chassis
set snmp trap-group test categories link
set snmp trap-group test categories routing
set snmp trap-group test categories startup
set snmp trap-group test categories services
set snmp trap-group test targets 192.168.10.198
set routing-options static route 0.0.0.0/0 next-hop 192.168.10.1
set routing-options static route 0.0.0.0/0 no-readvertise
set routing-options router-id 1.1.1.1
set protocols ospf reference-bandwidth 1g
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 passive
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0

[edit]
juliopdx@R1#
```

## Bonus: Automating the Configuration

I couldn’t help myself, I wanted to take a shot at automating some of the configuration during my Junos learning. I’ll keep this short and sweet but I stayed with the common tools of Nornir, NAPALM, and some neat Jinja2 templates. I have covered everything in the repository (linked below) before besides the Jinja2 templates. I’ll go over those and the script to close this one out.

### Junos Configuration Hierarchy and jinja2

The Junos text output I showed you initially in this post really lends itself to being automated. Even from a text standpoint. Everything has really natural breaks in separation. I mentioned this before but if you wanted to modify interfaces, this would all be done under the interfaces level. I took that same approach and broke the Jinja templates out. We have a main **base.j2** template that will then call on the rest. This will then build the entire running configuration of a device.

```shell
(venv) juliopdx@juliopdx-pop:~/git/junos-auto$ tree templates/
templates/
├── base.j2
├── interfaces.j2
├── protocols.j2
├── routing-options.j2
├── snmp.j2
└── system.j2
```

`base.j2`

```jinja2
version 18.2R1.9;
{% include 'system.j2' %}

{% include 'interfaces.j2' %}

{% include 'snmp.j2' %}

{% include 'routing-options.j2' %}

{% include 'protocols.j2' %}
```

`interfaces.j2`

```jinja2
interfaces {
{% for interface in host.info.interfaces %}
    {{ interface.name }} {
        unit {{ interface.unit }} {
            family {{ interface.family }} {
                address {{ interface.address }};
            }
        }
    }
{% endfor %}
}
```

### Defining Host Data

I kept it simple and used YAML files to store data about each node. Please note this is a very simple example. There are definitely more ways to improve this and remove some of the duplicate information.

```shell
(venv) juliopdx@juliopdx-pop:~/git/junos-auto$ tree host_vars/
host_vars/
├── R1.yaml
├── R2.yaml
└── R3.yaml
```

`R1.yaml`

```yaml
---
interfaces:
  - name: ge-0/0/0
    unit: 0
    family: inet
    address: 10.0.12.1/24
  - name: ge-0/0/1
    unit: 0
    family: inet
    address: 10.0.13.1/24
  - name: ge-0/0/2
    unit: 0
    family: inet
    address: 10.0.0.1/24
  - name: fxp0
    unit: 0
    family: inet
    address: 192.168.10.143/24
  - name: lo0
    unit: 0
    family: inet
    address: 1.1.1.1/32

routing_options:
  statics:
    - route: 0.0.0.0/0
      next_hop: 192.168.10.1
      no_readvertise: True
  router_id: 1.1.1.1

protocols:
  ospf:
    ref_band: 1g
    areas:
      0.0.0.0:
        interfaces:
          - name: ge-0/0/2.0
            passive: True
          - name: lo0
            passive: True
          - name: ge-0/0/0.0
          - name: ge-0/0/1.0
```

### Nornir and NAPALM

The script below was a mix of work I’ve done in the past and a great video by IPvZero (linked below). In the video John, demonstrates the use of the **load_yaml** and **template_file** methods in Nornir. The former will load YAML data that will then be assigned to the current host. This data can then be used by the **template_file** method to create configurations files for each host.

`junos_norn.py`

```python
"""Script used to interfact with some Junos gear"""
from nornir import InitNornir
from nornir_napalm.plugins.tasks import napalm_configure
from nornir_utils.plugins.functions import print_result
from nornir_utils.plugins.tasks.data import load_yaml
from nornir_jinja2.plugins.tasks import template_file


def load_vars(task):
    """Loads data to be added to each host from vars files"""
    my_vars = task.run(task=load_yaml, file=f"./host_vars/{task.host}.yaml")
    task.host["info"] = my_vars.result
    template_configurations(task)


def template_configurations(task):
    """Builds network templates"""
    temp = task.run(
        name="Building device configuration",
        task=template_file,
        path="templates",
        template="base.j2",
    )
    config = temp.result
    deploy_configurations(task, config)


def deploy_configurations(task, config):
    """Loads device configurations"""
    task.run(
        name=f"Configuring {task.host}!",
        task=napalm_configure,
        configuration=config,
        replace=True,
    )


def main():
    """Used to run all the things"""
    norn = InitNornir(config_file="configs/config.yaml", core={"raise_on_error": True})
    result = norn.run(task=load_vars)
    print_result(result)


if __name__ == "__main__":
    main()
```

## Outro and Links

Thank you all for reading this far. I hope you learned a bit about Junos and some automation. What I showed in this post barely scratches the surface on what Junos can do, definitely more to learn! This wasn’t created in a bubble and I will link some awesome resources I had when learning about Junos and automation. Please check it out if you get a chance. I will definitely be using Junos more in my day to day labbing and learning.

- [Junos for IOS Engineers by Chris Parker](https://www.networkfuntimes.com/new-series-a-guide-to-junos-for-ios-engineers/)
- [Juniper Networks JNCIA Course by Rich Bibby on Pluralsight](https://www.pluralsight.com/authors/rich-bibby)
- [Scrapli CFG Video by IPvZero](https://www.youtube.com/watch?v=xKe_KxT2doY)
- [GitHub Repository](https://github.com/JulioPDX/junos-auto)
