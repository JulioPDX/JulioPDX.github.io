---
title: "My Journey Learning About the Palo Alto Networks Python SDK"
date: 2021-11-22T13:18:29-08:00
draft: false
toc: true
images:
tags:
  - Python
  - PAN
---

## Introduction

Hello and thank you for checking out another blog post. This one will be all about Python and Palo Alto Networks (PAN). I was recently going through a PAN Firewall course on Pluralsight by Craig Stansbury. Craig does an excellent job of walking learners through the process of administering and securing a PAN firewall. I wanted to take this opportunity to double dip and try to wrap my head around the PAN pan-os-python library. This blog wont go into details of the course but it will go through the process of configuring the firewall using the PAN Python SDK.

## Object-Oriented Programming (OOP)

I will do my best to break down OOP but in case my explanations are not clear enough, I will link a great post by the pros at Real Python. The Palo SDK uses objects very heavily in their code base. For example, if you create an instance of a firewall object, this not only creates a firewall but will also inherit attributes and methods from a parent class. Please see my router class example below.

```python
class Router:
    pizza_box = True

    def __init__(self, vendor, os):
        self.vendor = vendor
        self.os = os

class CiscoRouter(Router):
    pass
```

```python
>>> my_router = CiscoRouter("Cisco", "IOS")
>>> my_router.vendor
'Cisco'
>>> my_router.os
'IOS'
>>> my_router.pizza_box
True
>>>
```

In the example above, we have a parent class called **Router** and a child of that class called **CiscoRouter**. We set “Cisco” and “IOS” as the vendor and os respectively. Even though we did not specify the “pizza_box” attribute in the **CiscoRouter** class, it was inherited from the **Router** parent class. Please view this as an oversimplification. There is so much more to learn in this arena and I have a long way to go.

## OOP in the PAN SDK

The PAN SDK works much in the same way. There are main parent classes and many child classes below them that will inherit attributes and available methods. Let me show you an example straight from the pan-os-python code base. Lets look at a firewall object.

```python
class Firewall(PanDevice):
    """A Palo Alto Networks Firewall
    This object can represent a firewall physical chassis,virtual firewall, or individual vsys.
    """
```

The **Firewall** class is actually a child class of the **PanDevice** class. Lets instantiate a firewall object to get us going.

```python
>>> fw = firewall.Firewall("192.168.10.192", "admin", "PaloAlto123!")
>>> type(fw)
<class 'panos.firewall.Firewall'>
>>> from rich import inspect
>>> inspect(fw)
╭──────────────────── <class 'panos.firewall.Firewall'> ─────────────────────╮
│ A Palo Alto Networks Firewall                                              │
│                                                                            │
│ ╭────────────────────────────────────────────────────────────────────────╮ │
│ │ <Firewall '192.168.10.192' None at 0x7ffa0c5e5ac0>                     │ │
│ ╰────────────────────────────────────────────────────────────────────────╯ │
│                                                                            │
│                   api_key = 'LUFRPT1hUEZ2K0w3UGFzMHdxQVcxcVJLZ3VzS1NKNmc9… │
│              CHILDMETHODS = ()                                             │
│                  children = []                                             │
│                CHILDTYPES = (                                              │
│                                 'device.AuthenticationProfile',            │
│                                 'device.AuthenticationSequence',           │
│                                 'device.Vsys',                             │
│                                 'device.VsysResources',                    │
│                                 'device.SystemSettings',                   │
│                                 'device.LogSettingsSystem',                │
│                                 'device.LogSettingsConfig',                │
│                                 'device.PasswordProfile',                  │
│                                 'device.Administrator',                    │
│                                 'device.Telemetry',                        │
│                                 'device.SnmpServerProfile',                │
│                                 'device.EmailServerProfile',               │
│                                 'device.LdapServerProfile',                │
│                                 'device.SyslogServerProfile',              │
│                                 'device.HttpServerProfile',                │
│                                 'ha.HighAvailability',                     │
│                                 'objects.AddressObject',                   │
│                                 'objects.AddressGroup',                    │
│                                 'objects.ServiceObject',                   │
│                                 'objects.ServiceGroup',                    │
│                                 'objects.Tag',                             │
│                                 'objects.ApplicationObject',               │
│                                 'objects.ApplicationGroup',                │
│                                 'objects.ApplicationFilter',               │
│                                 'objects.ApplicationContainer',            │
│                                 'objects.ScheduleObject',                  │
│                                 'objects.SecurityProfileGroup',            │
│                                 'objects.CustomUrlCategory',               │
│                                 'objects.LogForwardingProfile',            │
│                                 'objects.DynamicUserGroup',                │
│                                 'objects.Region',                          │
│                                 'objects.Edl',                             │
│                                 'policies.Rulebase',                       │
│                                 'network.EthernetInterface',               │
│                                 'network.AggregateInterface',              │
│                                 'network.LoopbackInterface',               │
│                                 'network.TunnelInterface',                 │
│                                 'network.VlanInterface',                   │
│                                 'network.Vlan',                            │
│                                 'network.VirtualRouter',                   │
│                                 'network.ManagementProfile',               │
│                                 'network.VirtualWire',                     │
│                                 'network.IkeGateway',                      │
│                                 'network.IpsecTunnel',                     │
│                                 'network.IpsecCryptoProfile',              │
│                                 'network.IkeCryptoProfile',                │
│                                 'network.GreTunnel',                       │
│                                 'network.Dhcp',                            │
│                                 'network.Zone'                             │
│                             )                                              │
╰────────────────────────────────────────────────────────────────────────────╯
>>>
```

Think of those **CHILDTYPES** as things you can associate with this firewall object. For now lets take a quick detour and look at the **PanDevice** class.

```python
class PanDevice(PanObject):
    """A Palo Alto Networks device
    The device can be of any type (currently supported devices are firewall, or panorama). The class handles common device functions that apply to all device types.
    Usually this class is not instantiated directly. It is the base class for a firewall.Firewall object or a panorama.Panorama object.
    """

    NAME = "hostname"

    def __init__(
        self,
        hostname,
        api_username=None,
        api_password=None,
        api_key=None,
        port=443,
        is_virtual=None,
        timeout=1200,
        interval=0.5,
        *args,
        **kwargs
    )
```

At this point even **PanDevice** is a child of another class, **PanObject**. When we created our firewall object we did not specify a timeout. Lets check the firewall and see if this was inherited.

```python
>>> fw.timeout
1200
```

Lets take one more detour and look at the **PanObject** class.

```python
class PanObject(object):
    """Base class for all package objects
    This class defines an object that can be placed in a tree to generate configuration."""
```

It looks like **PanObject** is the base for all objects. Now you might be asking… why do I care? When configuring a PAN firewall, almost everything has some type of reference or dependency for something else to exist. For example, I cant assign an interface to a virtual router named “DMZ” if that does not exist first. Another example, I cant assign address objects to an address group if the address objects don’t exist. Once I wrapped my head around the concept of inheritance and configuration trees, the SDK made much more sense.

`From pan-os-python readthedocs`

![Inheritance](/blog/images/inheritance-diagram.png)

## Configuring Zone Example

I love providing examples and its one of the best ways I learn. I’ll walk you through configuring a firewall zone and the configuration tree involved to get the correct sequences down. For reference below is an example configuration tree from the official pan-os-python documentation.

![Zone Build](/blog/images/zone-build.png)

```python
>>> from panos import firewall
>>> from panos import network
>>> fw = firewall.Firewall("192.168.10.192", "admin", "PaloAlto123!")
>>> lan_zone = network.Zone(
...     name="User LAN",
...     mode="layer3",
...     zone_profile="DefaultZoneProtectionProfile",
...     log_setting="Zone-Forwarding-Profile",
...     enable_packet_buffer_protection=True,
... )
>>> fw.add(lan_zone)
<Zone User LAN 0x7ffa0b8aa7f0>
>>> lan_zone.create()
>>>
```

Lets break that all down. We are importing a few required libraries from panos. One will be used to instantiate a firewall object and the other will be used to build a zone object. We then define any required parameters to the zone object. This is then added to the **fw** object using the **add** method. There are many methods available, for this proof of concept I heavily used the **create** method.

### Repeating Myself

Once I had the process down to configure zones, interfaces, or anything else available in the SDK, I started to get everything on paper. Eventually the zone portion of my script looked something like below…

```python
# Creating Zones on Firewall
lan_zone = network.Zone(
    name="User LAN",
    mode="layer3",
    zone_profile="DefaultZoneProtectionProfile",
    log_setting="Zone-Forwarding-Profile",
    enable_packet_buffer_protection=True,
)
fw.add(lan_zone)
lan_zone.create()


dc_zone = network.Zone(
    name="Datacenter",
    mode="layer3",
    zone_profile="DefaultZoneProtectionProfile",
    log_setting="Zone-Forwarding-Profile",
    enable_packet_buffer_protection=True,
)
fw.add(dc_zone)
dc_zone.create()


dmz_zone = network.Zone(
    name="DMZ",
    mode="layer3",
    zone_profile="DefaultZoneProtectionProfile",
    log_setting="Zone-Forwarding-Profile",
    enable_packet_buffer_protection=True,
)
fw.add(dmz_zone)
dmz_zone.create()


outside_zone = network.Zone(
    name="Outside",
    mode="layer3",
    zone_profile="OutsideZoneProtectionProfile",
    log_setting="Zone-Forwarding-Profile",
    enable_packet_buffer_protection=True,
)
fw.add(outside_zone)
outside_zone.create()
```

For every zone, the steps required to create them is all the same. This is true for other pieces of the firewall configuration; interfaces, tags, etc. Eventually my configuration script grew rather large and had repetition all over the place. I left that file (connect.py) in the original repository so individuals could see how something can be transformed to be a bit more manageable.

### Making Zones and Not Repeating Myself

I eventually turned all of those steps into a simple function. The function would take two to three parameters. Usually something like the firewall object and the data required to create a zone.

```python
from configs.zones import zones

def create_zone(fire, curent_zone):
    """Creates firewall zone and returns object"""
    set_zone = network.Zone(**curent_zone)
    fire.add(set_zone)
    set_zone.create()
    return set_zone


for zone in zones:
    create_zone(fw, zone)
```

We now have a new import, zones! I figured it was more manageable to break out the parameters into individual Python files. An example of the directory and zones.py file is below.

```shell
(venv) juliopdx@juliopdx-pop:~/git/pan-auto$ tree configs/
configs/
├── address_groups.py
├── address_objects.py
├── __init__.py
├── interfaces.py
├── nats.py
├── routing.py
├── security_policies.py
├── tags.py
└── zones.py
```

`zones.py`

```python
zones = [
    {
        "name": "User LAN",
        "mode": "layer3",
        "enable_packet_buffer_protection": True,
    },
    {
        "name": "Datacenter",
        "mode": "layer3",
        "enable_packet_buffer_protection": True,
    },
    {
        "name": "DMZ",
        "mode": "layer3",
        "enable_packet_buffer_protection": True,
    },
    {
        "name": "Outside",
        "mode": "layer3",
        "enable_packet_buffer_protection": True,
    },
    {
        "name": "Test",
        "mode": "layer3",
        "enable_packet_buffer_protection": True,
    },
]
```

In the previous example you may have seen the line below and been a bit confused (I was).

```python
set_zone = network.Zone(**curent_zone)
```

What does “**” mean??? In Python this is a known as Kwargs or keyword arguments. This allows us to pass in the dictionaries we created to our object call and reference the correct parameters. This has the benefit of decoupling the object from the data completely. Every zone can have as many or as little keyword arguments as necessary to create the object.

## Making Prints Pretty

![Pretty Firewall](/blog/images/pretty-firewall.png)

I’ve been sharing terminal output like this on twitter during this learning journey. This isn’t required by any means but if I’m on the terminal all the time, why not make it look good? I’ll break down one example of these tasks, the rest follow a very similar structure.

```python
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn

with Progress(
    SpinnerColumn("bouncingBall", speed=0.6),
    BarColumn(),
    TextColumn("[progress.percentage]{task.description} {task.percentage:>3.0f}%"),
) as progress:

    job1 = progress.add_task("[bright_green]Configuring Zones", total=len(zones))

    while not progress.finished:
        for zone in zones:
            create_zone(fw, zone)
            progress.update(job1, advance=1)
```

This portion of the script will use the Rich Python Library from Will McGugan, linked below! I’m importing anything required to build the progress bar output. This example was based off of the included example under Rich. I select a type of spinner to display and what text will be shown. I then add a task to `progress`. In this case `job1` has a total equal to length of the zones variable. We have five zones so this will equate to five. Once the for loop starts, it will advance the task up by one every time. Eventually this will hit 5 and `progress` will be finished.

## Outro and Links

Thank you all for reading this post. Really means a lot and I hope you found something in here useful. Initially working with this SDK was a pain in the rear, but after digging into the configuration trees and inheritance it all clicked. Looking through some source code never hurt either! Please note, what’s included here and in the repository linked below is just a subset of what is possible using these tools. I wish you all the best and happy automating!

- [PAN Pluralsight Course by Craig Stansbury](https://www.pluralsight.com/courses/deploy-administer-secure-palo-alto-firewalls)
- [PAN-OS Python SDK Documentation](https://pan-os-python.readthedocs.io/en/latest/readme.html)
- [Object-Oriented Programming (OOP) in Python 3 by Real Python/David Amos](https://realpython.com/python3-object-oriented-programming/#what-is-object-oriented-programming-in-python)
- [Rich Python Library by Will McGugan](https://rich.readthedocs.io/en/stable/introduction.html)
- [GitHub Repository](https://github.com/JulioPDX/pan-auto)
