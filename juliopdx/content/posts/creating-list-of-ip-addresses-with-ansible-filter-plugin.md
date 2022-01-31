---
title: "Creating List of IP Addresses With Ansible Filter Plugin"
date: 2021-02-02
draft: False
toc: True
images:
tags:
  - Ansible
  - Python
---

## Intro

I recently had a use case where I needed a list of IP addresses from a prefix. My first step was to go through Ansible documentation on current filters to see if something would match my need. Long story short, there wasn’t. At least, nothing I could find which honestly is probably a huge possibility, and in that case this post is pointless. Ansible does have IP address filters but I noticed most all of the examples assume you already have a list of IP addresses to pass to the filters.

I should probably explain what a filter does just in case the reader isn’t aware. Filters allow you to transform data from something to something else. The simplest example being, turn this lower case string to upper case. See example below.

```yaml
---
- name: Filter Test
  connection: local
  hosts: localhost
  gather_facts: no
  vars:
    my_variable: mylowercasestring

  tasks:
    - name: Turning lower case string to UPPER CASE
      debug:
        msg: "{{ my_variable | upper }}"

...
```

```bash
PLAY [Filter Test]
 TASK [Turning lower case string to UPPER CASE]
 ok: [localhost] => {
     "msg": "MYLOWERCASESTRING"
 }
```
As you can see, filters are pretty neat ways to transform data. So there I was trying to create a list of IP addresses from a prefix, with no end in sight, and Google-fu failing me. I’m sure there is some whacky ways to make it happen, like a jinja template and reading a file or using some kind of range/with sequence option. In the end I came up with the following python script. This can be placed in the /filter_plugins directory where your playbook is stored.

```python
# ip_list.py
#!/usr/bin/python

from netaddr import IPNetwork

def ip_stuff(prefix):
    ips = []

    for ip in IPNetwork(prefix):
        ips.append(str(ip))
    return ips

class FilterModule(object):
    def filters(self):
        return {'ip_list': ip_stuff}
```

Breaking down the code, we first import one library, I know some folks aren’t a fan of this on filters but oh well. Then we create an empty list called ips, we then trigger a for loop to each individual entry under the prefix that will be passed in. Each entry will be added to the ips list. After that is all done we return the final list. Shout out to the oddbit blog for giving me a place to start on creating this filter.

```yaml
# ip_list_test.yaml
---
- name: IP List Test
  connection: local
  hosts: localhost
  gather_facts: no
  vars:
    my_prefix: 10.10.10.0/29

  tasks:
    - name: Setting a fact for my list of IP addresses
      set_fact:
        my_ip_addresses: "{{ my_prefix | ip_list }}"

    - name: Print list of IP addresses
      debug:
        msg: "{{ my_ip_addresses }}"

...
```

```
PLAY [IP List Test] *
 TASK [Setting a fact for my list of IP addresses] ***
 ok: [localhost]
 TASK [Print list of IP addresses] *
 ok: [localhost] => {
     "msg": [
         "10.10.10.0",
         "10.10.10.1",
         "10.10.10.2",
         "10.10.10.3",
         "10.10.10.4",
         "10.10.10.5",
         "10.10.10.6",
         "10.10.10.7"
     ]
 }
```

That’s about it. I hope you all enjoyed this write up and now you can make a list of IP addresses from a prefix… woohoo! Thank you for reading this far.
