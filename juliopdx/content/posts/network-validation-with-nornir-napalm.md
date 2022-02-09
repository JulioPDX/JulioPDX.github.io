---
title: "Network Validation With Nornir & Napalm"
date: 2021-02-27
draft: false
toc: true
images:
  - /blog/images/nornir_logo_02.jpg
tags:
  - Python
  - Network Automation
  - Nornir
  - NAPALM
---

![Nornir Image](/blog/images/nornir_logo_02.jpg)

## Intro

I wanted to get my feet wet with [Nornir](https://nornir.readthedocs.io/en/latest/)/[NAPALM](https://napalm.readthedocs.io/en/latest/) and Python network automation in general. One simple goal to start with in network automation is validating configurations or deploying “show commands” and returning some kind of useful information. I wanted to start with something very simple, as easy wins make me want to keep progressing. In this post I will break down how I validate SNMP information on network devices using Nornir and NAPALM.

I think its worth knowing a bit of background to set the stage for future network engineers that are curious about diving into python for network automation. For starters, you can do it! I started my career with zero knowledge about python or programming in general. My initial and current automation efforts revolve mostly around using [Ansible](https://docs.ansible.com/ansible/latest/network/index.html). I think both tools are great and they each have their place in this space. In the bottom of this post I’ll include some links on resources I’ve used.

I wont go too deep into explaining Nornir and NAPALM but I’ll do my best. Nornir is a framework in which, it can provide a foundation for plugins or tools to be constructed around it. Think of Ansible but pure Python. NAPALM acts almost like a translator for multivendor environments. For example, if our goal is to get interface information or SNMP information from a device, we don’t care what commands are executed. We just need the data returned to use in a constructed manner.

NAPALM does this in a concept it calls “getters”. Getters are basically functions in NAPALM that interact with networking devices and return structured information. For a list of all supported vendors and getters, please see [here](https://napalm.readthedocs.io/en/latest/support/).

`Current Structure`

```bash
(venv) juliopdx@juliopdx-virtual-2004:~/git/nornir_snmp_validation$ tree -I venv
 .
 ├── config.yaml
 ├── inventory
 │   ├── defaults.yaml
 │   ├── groups.yaml
 │   └── hosts.yaml
 ├── nornir.log
 ├── README.md
 ├── requirements.txt
 ├── snmp_validate.py
 └── validate
     └── cisco.yaml
 2 directories, 9 files
```

```bash
# Creating python environment
python3 -m venv venv

# Activating virtual environment
source venv/bin/activate

# Clone git repo
git clone https://github.com/JulioPDX/nornir_snmp_validation.git
cd nornir_snmp_validation/

# Install requirements
pip install -r requirements.txt
```

Nornir was recently updated to split some of the functionality from Nornir core and the additional plugins. I just went ahead and installed pretty much all of the third party plugins at once. [Here is a link to those plugins](https://nornir.tech/nornir/plugins/).

```bash
 nornir==3.0.0
 nornir-ansible==2020.9.26
 nornir-jinja2==0.1.1
 nornir-napalm==0.1.1
 nornir-netbox==0.2.0
 nornir-netmiko==0.1.1
 nornir-pyez==0.0.10
 nornir-scrapli==2021.1.30
 nornir-utils==0.1.1
```

lets take a look at the config.yaml file. If you are coming from an Ansible background, just think about this as the ansible.cfg file. This allows you to set the inventory plugin, runner (# of ssh sessions and serial vs threaded), and locations for certain files that we’ll talk about in a bit.

`config.yaml`

```yaml
---
 inventory:
   plugin: SimpleInventory
   options:
     host_file: "inventory/hosts.yaml"
     group_file: "inventory/groups.yaml"
     defaults_file: "inventory/defaults.yaml"
 runner:
   plugin: threaded
   options:
     num_workers: 10
```

Now lets go into our inventory folder. You will notice that we have three files; hosts.yaml (for host data), group.yaml (group data), and defaults.yaml (defaults/global data). I found that Nornir is really flexible on where you can store information. I’ll show you what made sense to me.

```yaml
---
 port: 22
 username: cisco
 password: cisco
 platform: ios
# Since I am using only Cisco devices in this lab, I just set the
# login information at the defaults level.
# In multivendor deployments, you would probably break this out
# into the groups.yaml file or under each hosts...
```

At the moment I didn’t do much of anything with the goups.yaml file. In my testing it seems that if a group is called out in the hosts.yaml file, it must exist in the groups.yaml file.

`groups.yaml`

```yaml
---
cisco:
   data:
 switches:
   data:
 routers:
   data:
```

I tried to keep the host.yaml file as simple as possible. I created some fake data under each host to signify what site they belong to. This was mainly as an example from the official documentation. I assigned different hosts to different groups depending on if they were switches or routers, and what vendor. Oh, going back to the group variables. If different systems had different connection parameters, you could put that under the groups.yaml file and just assign the host to that group.

`hosts.yaml`

```yaml
---
NY-SW-01:
  hostname: 192.168.10.37
  groups:
    - cisco
    - switches
  data:
    site: NY
NE-RTR-01:
  hostname: 192.168.10.38
  groups:
    - cisco
    - routers
  data:
    site: NE
OR-SW-01:
  hostname: 192.168.10.36
  groups:
    - cisco
    - switches
  data:
    site: OR
```

I’m going to side step a bit here and get into NAPALM. NAPALM has a lot of great plugin tasks that work with Nornir but the two that I used in this test were napalm_get and napalm_validate. The reason I used the get task? I needed to see how the information was returned by NAPALM so that I could then build the source file to validate against. A seasoned professional could probably just go straight to the docs, but I needed a bit more. The following NAPALM docs helped in creating this. [Napalm Validate](https://napalm.readthedocs.io/en/develop/validate/index.html).

Below is a view of the /validate/cisco.yaml file. This was created from the link mentioned above and running the napalm_get task. Notice that the cisco.yaml file lists the name of the getter at the top or root of the file. You could lists multiple getters to validate multiple pieces of information, as long as its supported with your device.

`/validate/cisco.yaml`

```yaml
---
- get_snmp_information:
    community:
      myreadonly:
        acl: N/A
        mode: ro
      mysecurestring:
        acl: N/A
        mode: rw
    contact: JulioPDX
    location: mylocation
```

Here is a snippet from each devices SNMP configuration. Looking at the validate file above and the snippets, can you guess which device will fail?

`SNMP Configurations`

```bash
NY-SW-01(config)#do show run | inc snmp
 snmp-server community mysecurestring RW
 snmp-server community myreadonly RO
 snmp-server location mylocation
 snmp-server contact JulioPDX

NE-RTR-01(config)#do show run | inc snmp
 snmp-server community public RO
 snmp-server community private RW
 snmp-server location mylocation
 snmp-server contact JulioPDX

OR-SW-01(config)#do show run | inc snmp
 snmp-server community myreadonly RO
 snmp-server community mysecurestring RW
 snmp-server location mylocation
 snmp-server contact JulioPDX
```

`snmp_validate.py`

```python
# Import required modules
from nornir import InitNornir
from nornir_utils.plugins.functions import print_result
from nornir_napalm.plugins.tasks import napalm_get, napalm_validate

# Initializing Nornir
nr = InitNornir(config_file="config.yaml")

get_snmp = nr.run(name="GATHERING SNMP", task=napalm_get, getters=["get_snmp_information"])

print_result(get_snmp)

validate = nr.run(name="VALIDATE SNMP", task=napalm_validate, src="./validate/cisco.yaml")

print_result(validate)
```

Here is the output of the script, main parts to look at are towards the bottom when the snmp validations are printed to the screen.

```bash
(venv) juliopdx@juliopdx-virtual-2004:~/git/nornir_snmp_validation$ python snmp_validate.py
 GATHERING SNMP**
 NE-RTR-01 ** changed : False *
 vvvv GATHERING SNMP ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
 { 'get_snmp_information': { 'chassis_id': '',
                         'community': { 'private': { 'acl': 'N/A',
                                                     'mode': 'rw'},
                                        'public': { 'acl': 'N/A',
                                                    'mode': 'ro'}},
                         'contact': 'JulioPDX',
                         'location': 'mylocation'}}
 ^^^^ END GATHERING SNMP ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 NY-SW-01 ** changed : False
 vvvv GATHERING SNMP ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
 { 'get_snmp_information': { 'chassis_id': '9NA76XNH9Z8',
                         'community': { 'myreadonly': { 'acl': 'N/A',
                                                        'mode': 'ro'},
                                        'mysecurestring': { 'acl': 'N/A',
                                                            'mode': 'rw'}},
                         'contact': 'JulioPDX',
                         'location': 'mylocation'}}
 ^^^^ END GATHERING SNMP ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 OR-SW-01 ** changed : False
 vvvv GATHERING SNMP ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
 { 'get_snmp_information': { 'chassis_id': '9QA913OVFJS',
                         'community': { 'myreadonly': { 'acl': 'N/A',
                                                        'mode': 'ro'},
                                        'mysecurestring': { 'acl': 'N/A',
                                                            'mode': 'rw'}},
                         'contact': 'JulioPDX',
                         'location': 'mylocation'}}
 ^^^^ END GATHERING SNMP ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 VALIDATE SNMP***
 NE-RTR-01 ** changed : False *
 vvvv VALIDATE SNMP ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
 { 'complies': False,
 'get_snmp_information': { 'complies': False,
                         'extra': [],
                         'missing': [],
                         'present': { 'community': { 'complies': False,
                                                     'diff': { 'complies': False,
                                                               'extra': [],
                                                               'missing': [ 'myreadonly',
                                                                            'mysecurestring'],
                                                               'present': { }},
                                                     'nested': True},
                                      'contact': { 'complies': True,
                                                   'nested': False},
                                      'location': { 'complies': True,
                                                    'nested': False}}},
 'skipped': []}
 ^^^^ END VALIDATE SNMP ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 NY-SW-01 ** changed : False
 vvvv VALIDATE SNMP ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
 { 'complies': True,
 'get_snmp_information': { 'complies': True,
                         'extra': [],
                         'missing': [],
                         'present': { 'community': { 'complies': True,
                                                     'nested': True},
                                      'contact': { 'complies': True,
                                                   'nested': False},
                                      'location': { 'complies': True,
                                                    'nested': False}}},
 'skipped': []}
 ^^^^ END VALIDATE SNMP ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 OR-SW-01 ** changed : False
 vvvv VALIDATE SNMP ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
 { 'complies': True,
 'get_snmp_information': { 'complies': True,
                         'extra': [],
                         'missing': [],
                         'present': { 'community': { 'complies': True,
                                                     'nested': True},
                                      'contact': { 'complies': True,
                                                   'nested': False},
                                      'location': { 'complies': True,
                                                    'nested': False}}},
 'skipped': []}
 ^^^^ END VALIDATE SNMP ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 (venv) juliopdx@juliopdx-virtual-2004:~/git/nornir_snmp_validation$
```

Did you catch which one failed? If you guessed NE-RTR-01 earlier you would be correct. If you look at the output for the NE router, there is a key for “complies” and it is set to False. So now we know one device in our three device environment did not pass our SNMP validation check. This could then be tied to kick off another task to fix SNMP, report to monitoring, or some other workflow. All up to the imagination.

## Outro and Links

I hope this post has shown you how NAPALM and Nornir can be leveraged to validate device configurations and dive into a bit of network automation with Python. Not too bad right? Below you’ll find some links to resources I have used and a link to the git repo. There really are so many great resources in this space for network engineers. I cant possibly list them all. Thank you for reading this far, it is much appreciated! Stay safe out there.

- [Git Repository](https://github.com/JulioPDX/nornir_snmp_validation)
- [Python Crash Course 2nd Edition](https://nostarch.com/pythoncrashcourse2e)
- [Mastering Python Networking](https://www.packtpub.com/product/mastering-python-networking-third-edition/9781839214677)
- [Chuck Black 52 Weeks of Python](https://www.youtube.com/playlist?list=PLKZjLeG8AwtES8apo6gonH4XmN_9xIo8W)
- [NTC Awesome List](https://github.com/networktocode/awesome-network-automation)
