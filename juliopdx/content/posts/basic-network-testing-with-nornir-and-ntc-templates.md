---
title: "Basic Network Testing With Nornir and NTC Templates"
date: 2021-08-15T11:12:03-08:00
draft: false
toc: true
images:
  - /blog/images/ildefonso-polo.jpg
tags:
  - Cisco
  - Network Automation
---

## Introduction

Hello everyone and thank you for checking out another blog post. This time looking at testing the network. In my previous post, I mentioned testing could be a really neat and much needed addition to the code base. I had some spare time this weekend and couldn’t wait. Ill be honest, I consider myself a novice in python and programming in general. There is definitely many ways to accomplish a task and I’m sure there are plenty of more efficient ways to perform the actions I'll share with you.

## The Road to Network Testing

Initially I wanted to incorporate pytest for this weekend project. It generally made sense and experienced folks working in python use this to test anything from simple functions to extremely large code bases. I ran into a few issues when trying to incorporate pytest. Again, I’m fairly inexperienced in this realm and I’m sure a more experienced person would not run into these issues. The main issue I had with pytest was jobs failing after the first failed assert. Even when explicitly setting the *–maxfail* option. Overall I thought working with pytest was really fun and I cant wait to dig into it more.

Phase two of network testing had me working with Nornir and Netmiko to send a command, set `use_textfsm=True`, and then run a few if statements to check matching configuration. I'll include a portion of what that looked like below.

`Nornir and Netmiko`

```python
result = nornir.run(
    name="GET VRF",
    task=netmiko_send_command,
    command_string="show vrf",
    use_textfsm=True,
)

for host in nornir.inventory.hosts.keys():
    vrf_parsed = result[host].result
    list_of_vrf_values = [value for elem in vrf_parsed for value in elem.values()]

    print()

    for value in main_vrfs:
        if value in list_of_vrf_values:
            pretty(f"[bold blue]{value} VRF is configured on {host}[/bold blue]")
        else:
            pretty(f"[bold red]{value} VRF is not configured on {host}[/bold red]")
```

This approach generally worked but I had an issue in my lab. I’m not sure if it was a Netmiko issue or just my lab being weird and getting connection timeouts. In the end I decided to use Nornir with Scrapli to send the commands. Connections when using Scrapli seemed to just work, great job Carl! Once this was retrieved, I could then use the *parse_output* function from the *ntc_templates* library. Feel free to check out the repository with the test script (linked below).

## VRF Validation

`Imports`

```python
import os
import yaml
from rich import print as pretty
from nornir import InitNornir
from nornir_scrapli.tasks import send_command
from ntc_templates.parse import parse_output
```

The imports are nothing fancy. Just importing everything we will need for the script.

- os – Set environemt variables during script run
- yaml – Handle interaction with yaml stuffs
- rich – Because rich, thanks Will
- InitNornir- Run all the things for Nornir
- send_command – To send commands 🙂
- parse_output – Great library for turning output into structured data

`load yaml`

```python
main_vrfs = []

# Create list of VRFs defined in groups.yaml
with open("groups.yaml") as f:
    data = yaml.load(f, Loader=yaml.FullLoader)["pe"]["data"]["vrfs"]
    for vrf in data:
        main_vrfs.append(vrf["name"])
```

I thought this was pretty slick. We are opening the *groups.yaml* file seen from the previous post. This file lists all of our VRFs to be configured on the PE devices. Why not reuse it and save some typing? Once that is done we set the VRF portion of the yaml file to a variable called *data*. After this we loop over all the VRFs and add just the name to our empty list above. Hold that thought as this list will be used later in the script.

`test_vrf function intro`

```python
def test_vrf():

    """
    Main VRF test
    """
    os.environ[
        "NET_TEXTFSM"
    ] = "./venv/lib/python3.9/site-packages/ntc_templates/templates"

    nornir = InitNornir(config_file="config.yaml")

    result = nornir.run(
        name="GET VRF",
        task=send_command,
        command="show vrf",
    )
```

If any of you have worked with pytest, you can tell by the function naming that I was serious about incorporating it. I will in the future! Most of this has been shown in previous posts but one little difference is that *os.environ section*. When I was building this, the script couldn’t locate the *ntc_templates*. Essentially, we are just telling the script where to find the templates. You will probably have to update this for your environment.

`For all the hosts`

```python
    for host in nornir.inventory.hosts.keys():
        remote_vrfs = []
        vrf_parsed = parse_output(
            platform="cisco_ios", command="show vrf", data=result[host].result
        )

        # Looping over all parsed VRFs to build new list
        for remote_vrf in vrf_parsed:
            remote_vrfs.append(remote_vrf["name"])
```

I really love this about python and programming in general, what you see above is fairly basic and learned pretty early on in programming studies. But the power it can demonstrate is really neat to me. At this point in the script, all hosts have been connected to and information captured. We are using this initial for loop to loop over the data that was gathered from the *show vrf* command. We create an empty list called *remote_vrfs*, then run the *parse_output* function from the *ntc_templates* library to get data in a structured format. Once that is done we run another for loop to add VRF names gathered from PE routers to our previously empty list. At this point we now have two lists created. One for VRFs we want configured on devices and one from the actual VRFs already configured on the PE routers.

`Set Theory`

```python
        set_diff = set(main_vrfs) - set(remote_vrfs)

        # If a set difference is found, print the nice red output!
        if set_diff:
            pretty(
                f"[bold red]The following VRFs are not configured on {host}: {set_diff [/bold red]"
            )
        else:
            pretty(f"[bold blue]All VRF tests passed on router {host}[/bold blue]")
```

Good old set theory to the rescue. I learned this a while back from a course by… who other than Nick Russo. Discrete mathematics in high school may have helped but that was way too long ago. We take the two lists we have already created, then convert them to sets. From there we just look for the difference between *main_vrfs* set and the *remote_vrfs* set. Below is an example on the interpreter of what I hope to accomplish.

```python
>>> main_vrfs = {"CUSTOMER_777","CUSTOMER_789"}
>>> remote_vrfs = {"CUSTOMER_777","CUSTOMER_789","MGMT"}
>>> main_vrfs - remote_vrfs
set()
>>>
```

## Script Output

![VRF Test Passed](/blog/images/vrf_check_working.png)

![VRF Test Failed](/blog/images/vrf_check_not_working.png)

## Outro and Links

Thank you for staying this long. I hope you enjoyed the post and found something useful. As you can see I have a long way to go with network testing but it really is an important skill. I'll include some links below!

- [Featured Image by Ildefonso Polo](https://unsplash.com/photos/DX9X0g0Cg88)
- [GitHub Repository](https://github.com/JulioPDX/auto_mpls_l3vpn)
- [Nornir](https://nornir.readthedocs.io/en/latest/)
- [Scrapli](https://github.com/scrapli/nornir_scrapli)
- [NTC Templates](https://github.com/networktocode/ntc-templates)
- [Set Theory](https://www.geeksforgeeks.org/python-set-difference/)
