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
